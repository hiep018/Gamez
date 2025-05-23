<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Game Bắn Súng 3D - Cử Chỉ Tay (Sửa Lỗi Âm Thanh)</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tone/14.8.49/Tone.js" crossorigin="anonymous"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js" crossorigin="anonymous"></script>
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #0c0a09;
            color: #e5e7eb;
            display: flex;
            flex-direction: column;
            align-items: center;
            min-height: 100vh;
            margin: 0;
            padding: 1rem;
            overflow-x: hidden;
        }
        .container-wrapper {
            display: flex;
            flex-wrap: wrap;
            justify-content: center;
            gap: 1rem;
            width: 100%;
            max-width: 1380px;
        }
        .canvas-container, .game-container-3d {
            border: 1px solid #374151;
            border-radius: 0.5rem;
            padding: 0.5rem;
            background-color: #1c1917;
            box-shadow: 0 4px 6px rgba(0,0,0,0.2);
            display: flex;
            flex-direction: column;
            align-items: center;
            position: relative;
        }
        .canvas-container h2, .game-container-3d h2 {
            margin-top: 0;
            margin-bottom: 0.5rem;
            font-size: 1.1rem;
            color: #a8a29e;
        }
        #handCanvas {
            max-width: 100%;
            height: auto;
            background-color: #292524;
            border-radius: 0.375rem;
        }
        #gameCanvasContainer3D {
            width: 640px;
            height: 480px;
            max-width: 100%;
            border-radius: 0.375rem;
            overflow: hidden;
            border: 2px solid #059669;
            background-color: #000;
        }
        #gameCanvasContainer3D > canvas {
            display: block;
        }
        .controls, .game-info {
            background-color: #1c1917;
            padding: 1rem;
            border-radius: 0.5rem;
            margin-bottom: 1rem;
            text-align: center;
            width: 100%;
            max-width: 600px;
        }
        .controls button {
            background-color: #059669;
            color: white;
            padding: 0.5rem 1rem;
            border: none;
            border-radius: 0.375rem;
            cursor: pointer;
            margin: 0.25rem 0.5rem;
            transition: background-color 0.2s;
        }
        .controls button:hover {
            background-color: #047857;
        }
        .controls button:disabled {
            background-color: #44403c;
            cursor: not-allowed;
        }
        .loading-spinner {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            z-index: 100;
            border: 4px solid rgba(255, 255, 255, 0.2);
            border-radius: 50%;
            border-top-color: #60a5fa;
            width: 50px;
            height: 50px;
            animation: spin 1s linear infinite;
        }
        @keyframes spin {
            to { transform: translate(-50%, -50%) rotate(360deg); }
        }
        .message-box {
            position: fixed;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            background-color: #ef4444;
            color: white;
            padding: 0.75rem 1.25rem;
            border-radius: 0.375rem;
            box-shadow: 0 4px 6px rgba(0,0,0,0.2);
            z-index: 1000;
            display: none;
            text-align: center;
        }
        .gemini-status {
            font-size: 0.8rem;
            color: #78716c;
            margin-top: 0.5rem;
        }
        @media (max-width: 1024px) {
            .container-wrapper {
                flex-direction: column;
                align-items: center;
            }
            .canvas-container, .game-container-3d {
                width: 90%;
            }
            #gameCanvasContainer3D {
                width: 100%;
                height: 300px;
            }
             #handCanvas {
                width: 100% !important;
                height: auto !important;
            }
        }
    </style>
</head>
<body>
    <h1 class="text-3xl font-bold my-4 text-center text-green-500">Game Bắn Súng 3D - Cử Chỉ Tay (Sửa Lỗi Âm Thanh)</h1>

    <div class="controls">
        <button id="startButton" class="text-lg">Bắt đầu Game</button>
        <button id="resetButton" class="text-lg" disabled>Chơi lại</button>
        <button id="fetchTauntsButton" class="text-sm">✨ Lấy Lời Trêu Chọc Mới Từ AI</button>
        <p id="geminiStatus" class="gemini-status"></p>
    </div>

    <div class="game-info text-lg">
        <p>Điểm: <span id="scoreDisplay" class="font-semibold text-xl">0</span></p>
        <p>Đạn còn lại: <span id="ammoDisplay" class="font-semibold text-xl">∞</span></p>
        <p>Cử chỉ: <span id="gestureDisplay" class="font-semibold text-xl">Đang chờ...</span></p>
        <p class="mt-2">Mục tiêu nói: <span id="targetTauntDisplay" class="font-semibold text-md italic text-yellow-400">...</span></p>
        <p class="mt-1">AI Game: <span id="gameAiCommentDisplay" class="font-semibold text-md italic text-cyan-400">...</span></p>
    </div>

    <div class="container-wrapper">
        <div class="canvas-container">
            <h2>Camera & Nhận Diện Tay (2D)</h2>
            <div id="loadingSpinner" class="loading-spinner"></div>
            <video id="inputVideo" style="display: none;"></video>
            <canvas id="handCanvas" width="480" height="360"></canvas>
        </div>

        <div class="game-container-3d">
            <h2>Trò Chơi (3D)</h2>
            <div id="gameCanvasContainer3D"></div>
        </div>
    </div>
    <div id="messageBox" class="message-box"></div>

    <script type="module">
        // --- DOM Elements ---
        const videoElement = document.getElementById('inputVideo');
        const handCanvasElement = document.getElementById('handCanvas');
        const handCanvasCtx = handCanvasElement.getContext('2d');
        const gameCanvasContainer3D = document.getElementById('gameCanvasContainer3D');
        const startButton = document.getElementById('startButton');
        const resetButton = document.getElementById('resetButton');
        const fetchTauntsButton = document.getElementById('fetchTauntsButton');
        const scoreDisplay = document.getElementById('scoreDisplay');
        const ammoDisplay = document.getElementById('ammoDisplay');
        const gestureDisplay = document.getElementById('gestureDisplay');
        const targetTauntDisplay = document.getElementById('targetTauntDisplay');
        const gameAiCommentDisplay = document.getElementById('gameAiCommentDisplay');
        const loadingSpinner = document.getElementById('loadingSpinner');
        const messageBox = document.getElementById('messageBox');
        const geminiStatusElement = document.getElementById('geminiStatus');

        // --- MediaPipe Hands Setup ---
        const hands = new window.Hands({
            locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
        });
        hands.setOptions({
            maxNumHands: 1,
            modelComplexity: 0, // 0 for lite, 1 for full. Lite is faster.
            minDetectionConfidence: 0.6,
            minTrackingConfidence: 0.6
        });
        hands.onResults(onHandResults);

        const cameraUtils = new window.Camera(videoElement, {
            onFrame: async () => {
                if (videoElement.readyState >= videoElement.HAVE_CURRENT_DATA) { // More robust check
                    await hands.send({ image: videoElement });
                }
            },
            width: 480, height: 360
        });

        // --- Sound Effects (Tone.js) ---
        let shootSound, hitSound, gameOverSound, soundsReady = false;
        function setupSounds() {
            if (typeof Tone !== 'undefined') {
                shootSound = new Tone.Synth({ oscillator: { type: "triangle" }, envelope: { attack: 0.005, decay: 0.1, sustain: 0.05, release: 0.1 }, volume: -12 }).toDestination();
                hitSound = new Tone.NoiseSynth({ noise: { type: "white" }, envelope: { attack: 0.005, decay: 0.05, sustain: 0, release: 0.1 }, volume: -8 }).toDestination();
                gameOverSound = new Tone.Synth({ oscillator: { type: "sine" }, envelope: { attack: 0.01, decay: 0.5, sustain: 0.1, release: 0.5 }, volume: -10 }).toDestination();
                soundsReady = true;
            } else {
                console.warn("Tone.js not loaded. Sound effects will be unavailable.");
            }
        }
        setTimeout(setupSounds, 500);

        async function ensureAudioContextRunning() {
            if (typeof Tone !== 'undefined' && Tone.context.state !== 'running') {
                try {
                    await Tone.start();
                    // console.log("AudioContext started successfully by ensureAudioContextRunning.");
                } catch (e) {
                    console.warn("ensureAudioContextRunning: Tone.start() failed:", e);
                    // soundsReady = false; // Optionally disable sounds if Tone.start fails
                }
            }
        }


        // --- Three.js Scene Setup ---
        let scene, camera3D, renderer, shooter3D, ambientLight, directionalLight;
        const bullets3D = [];
        const targets3D = [];

        // Shared resources for performance
        let sharedBulletGeometry, sharedBulletMaterial;
        let particleInstancedMesh, particleGeometry, particleMaterial;
        const activeParticles = [];
        const MAX_PARTICLES = 500; // Max concurrent particles for explosions

        function initThreeJS() {
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x0A0A0D);

            const aspect = gameCanvasContainer3D.clientWidth / gameCanvasContainer3D.clientHeight;
            camera3D = new THREE.PerspectiveCamera(75, aspect, 0.1, 1000);
            camera3D.position.set(0, 2.5, 7);
            camera3D.lookAt(0, 1, -10);

            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(gameCanvasContainer3D.clientWidth, gameCanvasContainer3D.clientHeight);
            renderer.shadowMap.enabled = true;
            renderer.toneMapping = THREE.ACESFilmicToneMapping;
            renderer.toneMappingExposure = 1.0;
            gameCanvasContainer3D.innerHTML = '';
            gameCanvasContainer3D.appendChild(renderer.domElement);

            ambientLight = new THREE.AmbientLight(0x505060, 1.3);
            scene.add(ambientLight);

            directionalLight = new THREE.DirectionalLight(0xffffff, 1.8);
            directionalLight.position.set(5, 12, 8);
            directionalLight.castShadow = true;
            directionalLight.shadow.mapSize.width = 1024;
            directionalLight.shadow.mapSize.height = 1024;
            directionalLight.shadow.camera.near = 0.5;
            directionalLight.shadow.camera.far = 50;
            scene.add(directionalLight);

            // Shared Bullet Resources
            const bulletRadius = 0.025;
            const bulletHeight = 1.0;
            sharedBulletGeometry = new THREE.CylinderGeometry(bulletRadius, bulletRadius, bulletHeight, 8);
            sharedBulletMaterial = new THREE.MeshStandardMaterial({
                color: 0xfef9c3,
                emissive: 0xfde047,
                emissiveIntensity: 7.0,
                metalness: 0.0,
                roughness: 0.4
            });

            // Instanced Particle System Resources
            particleGeometry = new THREE.SphereGeometry(0.07, 6, 6); // Simple geometry for particles
            particleMaterial = new THREE.MeshBasicMaterial({
                transparent: true,
                opacity: 0.9, // Base opacity
                vertexColors: true // ESSENTIAL for per-instance colors via setColorAt
            });
            particleInstancedMesh = new THREE.InstancedMesh(particleGeometry, particleMaterial, MAX_PARTICLES);
            particleInstancedMesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage); // For frequent updates
            if (particleInstancedMesh.instanceColor) { // Check if instanceColor attribute exists
                 particleInstancedMesh.instanceColor.setUsage(THREE.DynamicDrawUsage);
            }
            scene.add(particleInstancedMesh);
            // Initialize particle instances (e.g., scale to 0 to hide)
            const dummyMatrix = new THREE.Matrix4();
            dummyMatrix.makeScale(0,0,0);
            for (let i = 0; i < MAX_PARTICLES; i++) {
                particleInstancedMesh.setMatrixAt(i, dummyMatrix);
            }
            particleInstancedMesh.instanceMatrix.needsUpdate = true;
        }

        // --- Gemini API Integration ---
        let targetTaunts = ["Bắn trượt rồi!", "Thử lại xem nào!", "Chậm thế!", "Dễ ợt!", "Ngắm cho chuẩn!"];
        let lastAiCommentTime = 0;
        const AI_COMMENT_INTERVAL = 450; // Frames

        // IMPORTANT: Replace with your actual API key or use a backend proxy for security.
        const GEMINI_API_KEY = ""; // "YOUR_GEMINI_API_KEY_HERE";

        async function callGeminiAPI(prompt, isJsonOutput = false) {
            geminiStatusElement.textContent = "AI đang suy nghĩ...";
            fetchTauntsButton.disabled = true;

            if (!GEMINI_API_KEY) {
                console.warn("Gemini API key is missing. AI features will be disabled.");
                showMessage("Chức năng AI chưa được cấu hình (thiếu API key).", true);
                geminiStatusElement.textContent = "Lỗi cấu hình AI.";
                if (isPlaying || !startButton.disabled) fetchTauntsButton.disabled = false;
                else fetchTauntsButton.disabled = true;
                return isJsonOutput ? [] : "Chức năng AI chưa được cấu hình.";
            }

            const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${GEMINI_API_KEY}`;
            const payload = { contents: [{ role: "user", parts: [{ text: prompt }] }] };

            if (isJsonOutput) {
                payload.generationConfig = {
                    responseMimeType: "application/json",
                    responseSchema: { type: "ARRAY", items: { type: "STRING" } }
                };
            }

            try {
                const response = await fetch(apiUrl, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });
                if (!response.ok) {
                    const errorBody = await response.text();
                    throw new Error(`Lỗi API: ${response.status}. Chi tiết: ${errorBody.substring(0, 150)}...`);
                }
                const result = await response.json();
                geminiStatusElement.textContent = "AI đã trả lời!";
                setTimeout(() => geminiStatusElement.textContent = "", 3000);

                if (result.candidates?.[0]?.content?.parts?.[0]?.text) {
                    const textData = result.candidates[0].content.parts[0].text;
                    return isJsonOutput ? JSON.parse(textData) : textData;
                } else {
                    throw new Error("Không nhận được nội dung hợp lệ từ AI.");
                }
            } catch (error) {
                console.error("Lỗi gọi Gemini API:", error);
                showMessage(`Lỗi AI: ${error.message}`, true);
                geminiStatusElement.textContent = "Lỗi AI.";
                return isJsonOutput ? [] : "AI đang gặp sự cố.";
            } finally {
                if (isPlaying || !startButton.disabled) fetchTauntsButton.disabled = false;
                else fetchTauntsButton.disabled = true;
            }
        }

        async function fetchNewTauntsFromAI() {
            const prompt = "Tạo một danh sách gồm 8 đến 12 lời trêu chọc ngắn (dưới 7 từ mỗi lời), hài hước và có chút thách thức mà một mục tiêu trong game bắn súng không gian 3D có thể nói với người chơi. Trả về dưới dạng một mảng JSON của các chuỗi. Ví dụ: [\"Bắn đi đâu thế?\", \"Còn non lắm!\"]";
            const newTaunts = await callGeminiAPI(prompt, true);
            if (Array.isArray(newTaunts) && newTaunts.length > 0) {
                targetTaunts = newTaunts;
                showMessage("Đã cập nhật danh sách lời trêu chọc từ AI!", false);
            } else {
                showMessage("Không thể lấy lời trêu chọc mới từ AI hoặc định dạng không đúng.", true);
            }
        }

        async function fetchGameAIComment(isGameOver = false) {
            let prompt;
            if (isGameOver) {
                prompt = `Bạn là một bình luận viên AI hài hước cho một game bắn súng 3D. Người chơi vừa kết thúc game với số điểm ${score}. Hãy đưa ra một bình luận ngắn (1-2 câu) về màn trình diễn của họ.`;
            } else {
                prompt = `Bạn là một bình luận viên AI hài hước cho một game bắn súng 3D. Người chơi hiện tại có ${score} điểm và đã chơi được ${Math.floor(gameTimer/60)} giây. Hãy đưa ra một bình luận ngắn gọn (1 câu, tối đa 15 từ) về tình hình hiện tại của người chơi. Ví dụ: "Tay súng cừ khôi!" hoặc "Cẩn thận kẻo hết đạn!"`;
            }
            const comment = await callGeminiAPI(prompt);
            gameAiCommentDisplay.textContent = comment;
        }

        // --- Game Logic Variables & Constants ---
        let score, ammo, gameTimer, isPlaying, gameOver, currentGestureAction;
        let handXNormalized = 0.5, handYNormalized = 0.5; // Default to center

        const SHOOTER_MOVE_RANGE_X = 6;
        const SHOOTER_MOVE_RANGE_Y = 2.5;
        const SHOOTER_BASE_Y = 1;
        const SHOOTER_MOVEMENT_SENSITIVITY = 0.5;
        const BULLET_SPEED_3D = 1.5;
        const TARGET_RADIUS_3D_MIN = 0.3;
        const TARGET_RADIUS_3D_MAX = 0.7;
        const INITIAL_TARGET_SPEED_Z_MIN = 0.065;
        const INITIAL_TARGET_SPEED_Z_MAX = 0.145;
        let currentTargetSpeedMin = INITIAL_TARGET_SPEED_Z_MIN;
        let currentTargetSpeedMax = INITIAL_TARGET_SPEED_Z_MAX;
        const TARGET_SPAWN_Z = -30;
        const TARGET_SPAWN_RANGE_X = 5;
        const TARGET_SPAWN_RANGE_Y_MIN = 0.5;
        const TARGET_SPAWN_RANGE_Y_MAX = 4.0;
        const INITIAL_TARGET_SPAWN_INTERVAL = 60; // Frames
        let currentTargetSpawnInterval = INITIAL_TARGET_SPAWN_INTERVAL;
        let frameCount = 0;
        let canShoot = true;
        const SHOOT_COOLDOWN_3D = 5; // Frames
        let shootCooldownTimer = 0;
        const GAME_DIFFICULTY_INCREASE_INTERVAL = 360; // Frames: Periodic difficulty increase
        const TARGET_SPEED_INCREASE_RATE = 0.006;       // How much speed increases periodically
        const TARGET_SPAWN_INTERVAL_DECREASE_RATE = 3; // How much spawn interval decreases periodically
        const MAX_TARGET_SPEED_Z_MAX_CAP = 0.30;
        const MIN_TARGET_SPAWN_INTERVAL_CAP = 15; // Smallest spawn interval (higher number = slower min spawn rate)

        // Hằng số để tăng độ khó khi bắn trúng
        const TARGET_SPEED_INCREASE_ON_HIT = 0.0015; // Tăng tốc độ mục tiêu một chút khi bắn trúng
        const TARGET_SPAWN_INTERVAL_DECREASE_ON_HIT = 0.75; // Giảm nhẹ khoảng thời gian xuất hiện mục tiêu khi bắn trúng

        const PARTICLE_LIFE_BASE = 20;
        const EXPLOSION_PARTICLE_COUNT = 20;
        const Y_CURL_THRESHOLD = 0.025;

        // --- Utility Functions ---
        function showMessage(messageText, isError = false) {
            messageBox.textContent = messageText;
            messageBox.style.backgroundColor = isError ? '#ef4444' : (messageText.toLowerCase().includes("game over") ? '#ef4444' : '#3b82f6');
            messageBox.style.display = 'block';
            setTimeout(() => { messageBox.style.display = 'none'; }, 3000);
        }

        // --- Hand Gesture Recognition ---
        function recognizeGameGesture(landmarks) {
            if (!landmarks || landmarks.length < 21) return 'AIM_ONLY';
            const getTip = (idx) => landmarks[idx];
            const getPip = (idx) => landmarks[idx - 2];

            const isIndexCurled = getTip(8).y > (getPip(8).y + Y_CURL_THRESHOLD);
            const isMiddleCurled = getTip(12).y > (getPip(12).y + Y_CURL_THRESHOLD);
            const isRingCurled = getTip(16).y > (getPip(16).y + Y_CURL_THRESHOLD);
            const isPinkyCurled = getTip(20).y > (getPip(20).y + Y_CURL_THRESHOLD);
            const isThumbCurled = getTip(4).y > (landmarks[3].y + Y_CURL_THRESHOLD / 2);

            if (isIndexCurled && isMiddleCurled && isRingCurled && isPinkyCurled && isThumbCurled) {
                return 'SHOOT';
            }
            return 'AIM_ONLY';
        }

        function onHandResults(results) {
            if (!handCanvasCtx || !handCanvasElement) return;
            loadingSpinner.style.display = 'none';
            handCanvasCtx.save();
            handCanvasCtx.clearRect(0, 0, handCanvasElement.width, handCanvasElement.height);
            handCanvasCtx.drawImage(results.image, 0, 0, handCanvasElement.width, handCanvasElement.height);

            currentGestureAction = 'AIM_ONLY';
            let gestureText = "Đang chờ...";

            if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                const handLandmarks = results.multiHandLandmarks[0];
                window.drawConnectors(handCanvasCtx, handLandmarks, window.Hands.HAND_CONNECTIONS, { color: '#00FF00', lineWidth: 3 });
                window.drawLandmarks(handCanvasCtx, handLandmarks, { color: '#FF0000', lineWidth: 1, radius: 3 });

                currentGestureAction = recognizeGameGesture(handLandmarks);

                if (handLandmarks[0] && typeof handLandmarks[0].x === 'number' && typeof handLandmarks[0].y === 'number') {
                    handXNormalized = handLandmarks[0].x;
                    handYNormalized = handLandmarks[0].y;
                } else {
                    handXNormalized = 0.5; handYNormalized = 0.5;
                }
                gestureText = (currentGestureAction === 'SHOOT') ? "BẮN (Nắm Tay)" : "Di chuyển tay để nhắm";
            } else {
                gestureText = "Không phát hiện tay";
                handXNormalized = 0.5; handYNormalized = 0.5;
                currentGestureAction = 'AIM_ONLY';
            }
            gestureDisplay.textContent = gestureText;
            handCanvasCtx.restore();
        }

        // --- Game Initialization and State Management ---
        function initGame3D() {
            bullets3D.forEach(b => { scene.remove(b.mesh); });
            bullets3D.length = 0;
            targets3D.forEach(t => { scene.remove(t.mesh); t.mesh.geometry.dispose(); t.mesh.material.dispose(); });
            targets3D.length = 0;

            activeParticles.forEach(p => p.active = false);
            const dummyMatrix = new THREE.Matrix4().makeScale(0,0,0);
            for (let i = 0; i < MAX_PARTICLES; i++) {
                particleInstancedMesh.setMatrixAt(i, dummyMatrix);
                if (particleInstancedMesh.instanceColor) {
                     particleInstancedMesh.setColorAt(i, new THREE.Color(0x000000));
                }
            }
            if (particleInstancedMesh.instanceMatrix) particleInstancedMesh.instanceMatrix.needsUpdate = true;
            if (particleInstancedMesh.instanceColor) particleInstancedMesh.instanceColor.needsUpdate = true;
            activeParticles.length = 0;

            if (shooter3D) scene.remove(shooter3D);
            shooter3D = new THREE.Group();
            const bodyMat = new THREE.MeshStandardMaterial({ color: 0x38bdf8, metalness: 0.6, roughness: 0.3 });
            const bodyGeo = new THREE.ConeGeometry(0.4, 1.0, 16);
            const shipBody = new THREE.Mesh(bodyGeo, bodyMat);
            shipBody.rotation.x = -Math.PI / 2; shipBody.position.y = 0.1;
            shooter3D.add(shipBody);
            const wingMat = new THREE.MeshStandardMaterial({ color: 0xa1a1aa, metalness: 0.7, roughness: 0.4 });
            const wingShape = new THREE.Shape().moveTo(0,0.3).lineTo(0.8,0).lineTo(0,-0.3).lineTo(0,0.3);
            const wingExtrudeSettings = { depth: 0.08, bevelEnabled: false };
            const wingGeo = new THREE.ExtrudeGeometry(wingShape, wingExtrudeSettings);
            const lWing = new THREE.Mesh(wingGeo, wingMat);
            lWing.rotation.y = Math.PI/2; lWing.position.set(-0.3, 0.1, -0.2); shooter3D.add(lWing);
            const rWing = new THREE.Mesh(wingGeo, wingMat);
            rWing.rotation.y = -Math.PI/2; rWing.position.set(0.3, 0.1, -0.2); shooter3D.add(rWing);
            const cockpitGeo = new THREE.SphereGeometry(0.15,16,16);
            const cockpitMat = new THREE.MeshStandardMaterial({color:0x0ea5e9, emissive:0x0284c7, transparent:true, opacity:0.75});
            const cockpit = new THREE.Mesh(cockpitGeo, cockpitMat);
            cockpit.position.set(0,0.15,-0.3); shooter3D.add(cockpit);
            shooter3D.position.set(0, SHOOTER_BASE_Y, 0);
            shooter3D.castShadow = shipBody.castShadow = lWing.castShadow = rWing.castShadow = true;
            scene.add(shooter3D);
            shooter3D.collisionRadius = 0.5;

            score = 0;
            gameTimer = 0; frameCount = 0; lastAiCommentTime = 0;
            isPlaying = false; gameOver = false; currentGestureAction = 'AIM_ONLY';
            handXNormalized = 0.5; handYNormalized = 0.5;
            canShoot = true; shootCooldownTimer = 0;
            targetTauntDisplay.textContent = "..."; gameAiCommentDisplay.textContent = "Chúc may mắn!";
            currentTargetSpeedMin = INITIAL_TARGET_SPEED_Z_MIN;
            currentTargetSpeedMax = INITIAL_TARGET_SPEED_Z_MAX;
            currentTargetSpawnInterval = INITIAL_TARGET_SPAWN_INTERVAL;
            updateUIDisplay();
            if(renderer && scene && camera3D) renderer.render(scene, camera3D);
        }

        async function startGame3D() {
            if (isPlaying) return;
            await ensureAudioContextRunning(); // Đảm bảo AudioContext chạy

            initGame3D();
            isPlaying = true; gameOver = false;
            startButton.disabled = true; resetButton.disabled = false;
            fetchTauntsButton.disabled = !GEMINI_API_KEY;
            showMessage("Game 3D bắt đầu!", false);
            gameLoop3D();
        }

        async function triggerGameOver3D(message = "Bạn đã bị bắn trúng!") {
            if (gameOver) return;
            gameOver = true; isPlaying = false;
            fetchTauntsButton.disabled = true;
            await ensureAudioContextRunning();
            if (soundsReady && gameOverSound && Tone.context.state === 'running') {
                try { gameOverSound.triggerAttackRelease("C3", "1s"); }
                catch(e) { console.error("GameOver sound error:", e); }
            }
            if (GEMINI_API_KEY) fetchGameAIComment(true);
            showMessage(`Game Over! ${message} Điểm của bạn: ${score}`, true);
        }

        function resetGame3D() {
            isPlaying = false; gameOver = true;
            startButton.disabled = false; resetButton.disabled = true;
            fetchTauntsButton.disabled = true;
            initGame3D();
        }

        function updateUIDisplay() {
            scoreDisplay.textContent = score;
            ammoDisplay.textContent = "∞";
        }

        // --- Game Object Updates & Interactions ---
        function updateShooter3D() {
            if (!shooter3D || !isPlaying) return;
            const targetX = (0.5 - handXNormalized) * 2 * SHOOTER_MOVE_RANGE_X;
            const targetY = SHOOTER_BASE_Y + (0.5 - handYNormalized) * 2 * SHOOTER_MOVE_RANGE_Y;

            shooter3D.position.x += (targetX - shooter3D.position.x) * SHOOTER_MOVEMENT_SENSITIVITY;
            shooter3D.position.y += (targetY - shooter3D.position.y) * SHOOTER_MOVEMENT_SENSITIVITY;

            shooter3D.position.x = Math.max(-SHOOTER_MOVE_RANGE_X, Math.min(shooter3D.position.x, SHOOTER_MOVE_RANGE_X));
            shooter3D.position.y = Math.max(SHOOTER_BASE_Y - SHOOTER_MOVE_RANGE_Y, Math.min(shooter3D.position.y, SHOOTER_BASE_Y + SHOOTER_MOVE_RANGE_Y));
        }

        async function shootBullet3D() {
            if (!canShoot || !shooter3D || !isPlaying) return;
            await ensureAudioContextRunning();
            const bullet = new THREE.Mesh(sharedBulletGeometry, sharedBulletMaterial);
            const localOffset = new THREE.Vector3(0, 0.1, -0.6);
            bullet.position.copy(shooter3D.localToWorld(localOffset.clone()));
            const velocity = new THREE.Vector3(0, 0, -1).applyQuaternion(shooter3D.quaternion).normalize();
            bullet.lookAt(bullet.position.clone().add(velocity));
            bullet.rotateX(Math.PI / 2);
            scene.add(bullet);
            bullets3D.push({ mesh: bullet, velocity: velocity.multiplyScalar(BULLET_SPEED_3D) });
            canShoot = false; shootCooldownTimer = SHOOT_COOLDOWN_3D;
            if (soundsReady && shootSound && Tone.context.state === 'running') {
                 try { shootSound.triggerAttackRelease("A5", "0.04s"); }
                 catch(e) { console.error("Shoot sound error:", e); }
            }
        }

        function updateBullets3D() {
            for (let i = bullets3D.length - 1; i >= 0; i--) {
                const b = bullets3D[i];
                b.mesh.position.add(b.velocity);
                if (b.mesh.position.z < TARGET_SPAWN_Z - 20 || Math.abs(b.mesh.position.x) > SHOOTER_MOVE_RANGE_X + 15 || Math.abs(b.mesh.position.y) > SHOOTER_MOVE_RANGE_Y + 15) {
                    scene.remove(b.mesh); bullets3D.splice(i, 1);
                }
            }
            if (!canShoot) {
                shootCooldownTimer--;
                if (shootCooldownTimer <= 0) canShoot = true;
            }
        }

        function addTarget3D() {
            const radius = Math.random() * (TARGET_RADIUS_3D_MAX - TARGET_RADIUS_3D_MIN) + TARGET_RADIUS_3D_MIN;
            const geo = new THREE.SphereGeometry(radius, 16, 16);
            const mat = new THREE.MeshStandardMaterial({
                color: new THREE.Color(`hsl(${Math.random() * 360}, 85%, 65%)`),
                metalness: 0.2, roughness: 0.6
            });
            const target = new THREE.Mesh(geo, mat);
            target.position.set(
                (Math.random() - 0.5) * 2 * TARGET_SPAWN_RANGE_X,
                (Math.random()*(TARGET_SPAWN_RANGE_Y_MAX - TARGET_SPAWN_RANGE_Y_MIN)) + TARGET_SPAWN_RANGE_Y_MIN,
                TARGET_SPAWN_Z
            );
            target.castShadow = true;
            const taunt = targetTaunts[Math.floor(Math.random() * targetTaunts.length)];
            scene.add(target);
            targets3D.push({
                mesh: target, radius: radius, taunt: taunt,
                speedZ: Math.random() * (currentTargetSpeedMax - currentTargetSpeedMin) + currentTargetSpeedMin
            });
        }

        function updateTargets3D() {
            let closestTaunt = "..."; let minZ = Infinity;
            for (let i = targets3D.length - 1; i >= 0; i--) {
                const t = targets3D[i];
                t.mesh.position.z += t.speedZ;
                if (t.mesh.position.z > camera3D.position.z + 5) {
                    scene.remove(t.mesh); t.mesh.geometry.dispose(); t.mesh.material.dispose();
                    targets3D.splice(i, 1);
                    score = Math.max(0, score - 5);
                } else if (shooter3D && t.mesh.position.z < minZ && t.mesh.position.z > shooter3D.position.z) {
                    minZ = t.mesh.position.z; closestTaunt = t.taunt;
                }
            }
            targetTauntDisplay.textContent = targets3D.length > 0 ? closestTaunt : "...";
        }

        function createExplosion3D(position, explosionColor) {
            for (let k = 0; k < EXPLOSION_PARTICLE_COUNT; k++) {
                let pData = activeParticles.find(p => !p.active);
                let pIndex = -1;

                if (pData) {
                    pIndex = pData.id;
                } else if (activeParticles.length < MAX_PARTICLES) {
                    pIndex = activeParticles.length;
                    pData = { id: pIndex };
                    activeParticles.push(pData);
                } else { continue; }

                pData.active = true;
                pData.position = position.clone();
                pData.velocity = new THREE.Vector3((Math.random()-0.5)*0.3, (Math.random()-0.5)*0.3, (Math.random()-0.5)*0.3);
                pData.life = PARTICLE_LIFE_BASE + Math.random() * 10;
                pData.baseScale = 0.5 + Math.random() * 0.8;
                pData.color = explosionColor.clone();

                const matrix = new THREE.Matrix4();
                matrix.setPosition(pData.position);
                particleInstancedMesh.setMatrixAt(pIndex, matrix);
                if (particleInstancedMesh.instanceColor) {
                    particleInstancedMesh.setColorAt(pIndex, pData.color);
                }
            }
            if (particleInstancedMesh.instanceMatrix) particleInstancedMesh.instanceMatrix.needsUpdate = true;
            if (particleInstancedMesh.instanceColor) particleInstancedMesh.instanceColor.needsUpdate = true;
        }

        function updateParticles3D() {
            let matrixNeedsUpdate = false;
            for (const p of activeParticles) {
                if (!p.active) continue;
                p.position.add(p.velocity);
                p.life--;

                const tempMatrix = new THREE.Matrix4();
                if (p.life <= 0) {
                    p.active = false;
                    tempMatrix.makeScale(0, 0, 0);
                } else {
                    const scaleProgress = p.life / (PARTICLE_LIFE_BASE + 5);
                    const currentScale = p.baseScale * Math.max(0, scaleProgress);
                    tempMatrix.makeScale(currentScale, currentScale, currentScale);
                    tempMatrix.setPosition(p.position);
                }
                particleInstancedMesh.setMatrixAt(p.id, tempMatrix);
                matrixNeedsUpdate = true;
            }
            if (matrixNeedsUpdate && particleInstancedMesh.instanceMatrix) particleInstancedMesh.instanceMatrix.needsUpdate = true;
        }

        async function checkCollisions() { // THAY ĐỔI: Thêm async
            if (!isPlaying) return;
            // Bullet-Target Collisions
            for (let i = bullets3D.length - 1; i >= 0; i--) {
                const bullet = bullets3D[i];
                const bulletRadiusEffective = sharedBulletGeometry.parameters.height / 2 * 0.4;
                for (let j = targets3D.length - 1; j >= 0; j--) {
                    const target = targets3D[j];
                    if (bullet.mesh.position.distanceTo(target.mesh.position) < target.radius + bulletRadiusEffective) {
                        createExplosion3D(target.mesh.position.clone(), target.mesh.material.color);
                        await ensureAudioContextRunning(); // THAY ĐỔI: Đảm bảo context chạy
                        if (soundsReady && hitSound && Tone.context.state === 'running') {
                           try {
                                hitSound.triggerAttackRelease("G#4", "0.05s");
                           } catch (e) { console.error("Hit sound error:", e); }
                        }
                        scene.remove(bullet.mesh); bullets3D.splice(i, 1);
                        scene.remove(target.mesh); target.mesh.geometry.dispose(); target.mesh.material.dispose();
                        targets3D.splice(j, 1);
                        score += 20;

                        // KÍCH HOẠT LẠI TÍNH NĂNG TĂNG TỐC ĐỘ KHI BẮN TRÚNG
                        // console.log(`HIT! Before update: SpeedMin=${currentTargetSpeedMin.toFixed(4)}, SpeedMax=${currentTargetSpeedMax.toFixed(4)}, Interval=${currentTargetSpawnInterval.toFixed(2)}`);

                        let newSpeedMin = parseFloat(currentTargetSpeedMin) + TARGET_SPEED_INCREASE_ON_HIT;
                        let newSpeedMax = parseFloat(currentTargetSpeedMax) + TARGET_SPEED_INCREASE_ON_HIT;
                        let newSpawnInterval = parseFloat(currentTargetSpawnInterval) - TARGET_SPAWN_INTERVAL_DECREASE_ON_HIT;

                        currentTargetSpeedMin = Math.min(newSpeedMin, MAX_TARGET_SPEED_Z_MAX_CAP * 0.95);
                        currentTargetSpeedMax = Math.min(newSpeedMax, MAX_TARGET_SPEED_Z_MAX_CAP);
                        currentTargetSpawnInterval = Math.max(newSpawnInterval, MIN_TARGET_SPAWN_INTERVAL_CAP);

                        if (isNaN(currentTargetSpeedMin)) currentTargetSpeedMin = INITIAL_TARGET_SPEED_Z_MIN;
                        if (isNaN(currentTargetSpeedMax)) currentTargetSpeedMax = INITIAL_TARGET_SPEED_Z_MAX;
                        if (isNaN(currentTargetSpawnInterval) || currentTargetSpawnInterval <= 0) {
                            currentTargetSpawnInterval = MIN_TARGET_SPAWN_INTERVAL_CAP;
                        }
                        
                        // console.log(`HIT! After update: SpeedMin=${currentTargetSpeedMin.toFixed(4)}, SpeedMax=${currentTargetSpeedMax.toFixed(4)}, Interval=${currentTargetSpawnInterval.toFixed(2)}`);

                        break; 
                    }
                }
            }

            // Target-Shooter Collisions
            if (!shooter3D || !shooter3D.collisionRadius || gameOver) return;
            for (let i = targets3D.length - 1; i >= 0; i--) {
                const target = targets3D[i];
                if (shooter3D.position.distanceTo(target.mesh.position) < target.radius + shooter3D.collisionRadius) {
                    createExplosion3D(shooter3D.position.clone(), new THREE.Color(0xffffff));
                    createExplosion3D(target.mesh.position.clone(), target.mesh.material.color);
                    scene.remove(target.mesh); target.mesh.geometry.dispose(); target.mesh.material.dispose();
                    targets3D.splice(i, 1);
                    await triggerGameOver3D("Tàu của bạn đã bị phá hủy!"); // THAY ĐỔI: await
                    return;
                }
            }
        }

        // --- Main Game Loop ---
        async function gameLoop3D() { // THAY ĐỔI: Thêm async
            if (gameOver) {
                if(renderer && scene && camera3D) renderer.render(scene, camera3D);
                return;
            }
            if (!isPlaying) {
                if(renderer && scene && camera3D) renderer.render(scene, camera3D);
                requestAnimationFrame(gameLoop3D);
                return;
            }

            frameCount++; gameTimer++;

            // Periodic Difficulty Scaling
            if (frameCount % GAME_DIFFICULTY_INCREASE_INTERVAL === 0 && frameCount > 0) {
                let periodicSpeedMin = parseFloat(currentTargetSpeedMin) + TARGET_SPEED_INCREASE_RATE;
                let periodicSpeedMax = parseFloat(currentTargetSpeedMax) + TARGET_SPEED_INCREASE_RATE;
                let periodicSpawnInterval = parseFloat(currentTargetSpawnInterval) - TARGET_SPAWN_INTERVAL_DECREASE_RATE;

                currentTargetSpeedMin = Math.min(periodicSpeedMin, MAX_TARGET_SPEED_Z_MAX_CAP * 0.9);
                currentTargetSpeedMax = Math.min(periodicSpeedMax, MAX_TARGET_SPEED_Z_MAX_CAP);
                currentTargetSpawnInterval = Math.max(periodicSpawnInterval, MIN_TARGET_SPAWN_INTERVAL_CAP);

                if (isNaN(currentTargetSpeedMin)) currentTargetSpeedMin = INITIAL_TARGET_SPEED_Z_MIN;
                if (isNaN(currentTargetSpeedMax)) currentTargetSpeedMax = INITIAL_TARGET_SPEED_Z_MAX;
                if (isNaN(currentTargetSpawnInterval) || currentTargetSpawnInterval <= 0) {
                    currentTargetSpawnInterval = MIN_TARGET_SPAWN_INTERVAL_CAP;
                }
            }

            await shootBullet3D(); // THAY ĐỔI: await

            updateShooter3D();
            
            const flooredInterval = Math.floor(currentTargetSpawnInterval);
            if (flooredInterval > 0 && frameCount % flooredInterval === 0) {
                addTarget3D();
            } else if (flooredInterval <= 0 && frameCount % MIN_TARGET_SPAWN_INTERVAL_CAP === 0 ) {
                 addTarget3D();
            }


            updateTargets3D();
            updateBullets3D();
            updateParticles3D();
            await checkCollisions(); // THAY ĐỔI: await
            updateUIDisplay();

            if (frameCount - lastAiCommentTime > AI_COMMENT_INTERVAL && targets3D.length > 0 && !gameOver && GEMINI_API_KEY) {
                fetchGameAIComment(false);
                lastAiCommentTime = frameCount;
            }

            if (!gameOver && targets3D.length > 15) {
                await triggerGameOver3D("Quá nhiều mục tiêu áp đảo!"); // THAY ĐỔI: await
            }

            if(renderer && scene && camera3D) renderer.render(scene, camera3D);
            if (!gameOver) requestAnimationFrame(gameLoop3D);
        }

        // --- Event Listeners & Initialization ---
        startButton.addEventListener('click', startGame3D);
        resetButton.addEventListener('click', resetGame3D);
        fetchTauntsButton.addEventListener('click', fetchNewTauntsFromAI);
        fetchTauntsButton.disabled = true;

        window.addEventListener('load', () => {
            initThreeJS();
            initGame3D();
            cameraUtils.start()
                .then(() => {
                    loadingSpinner.style.display = 'none';
                    showMessage("Camera sẵn sàng. Nhấn 'Bắt đầu Game'.", false);
                })
                .catch(err => {
                    console.error("Lỗi Camera:", err);
                    loadingSpinner.style.display = 'none';
                    let msg = "Không thể truy cập camera. Kiểm tra quyền và đảm bảo không có ứng dụng khác đang dùng.";
                    if (err.name === "NotFoundError") msg = "Không tìm thấy camera.";
                    else if (err.name === "NotAllowedError") msg = "Chưa cấp quyền camera. Kiểm tra cài đặt trình duyệt.";
                    else if (err.name === "NotReadableError") msg = "Lỗi đọc camera: Camera có thể đang được dùng bởi ứng dụng khác hoặc driver có vấn đề.";
                    showMessage(msg, true);
                    gestureDisplay.textContent = "Lỗi Camera";
                    startButton.disabled = true; fetchTauntsButton.disabled = true;
                });
            setTimeout(resizeGameCanvases, 100);
        });

        function resizeGameCanvases() {
            const handCanvasAR = 480 / 360;
            if (handCanvasElement.parentElement) {
                let newW = Math.min(handCanvasElement.parentElement.clientWidth - 16, videoElement.videoWidth || 480);
                handCanvasElement.width = newW;
                handCanvasElement.height = newW / handCanvasAR;
            }
            if (renderer && camera3D && gameCanvasContainer3D) {
                const w = gameCanvasContainer3D.clientWidth;
                const h = gameCanvasContainer3D.clientHeight;
                if (w > 0 && h > 0) {
                    renderer.setSize(w, h);
                    camera3D.aspect = w / h;
                    camera3D.updateProjectionMatrix();
                }
                if (!isPlaying && renderer && scene && camera3D) renderer.render(scene, camera3D);
            }
        }
        window.addEventListener('resize', resizeGameCanvases);

    </script>
</body>
</html>
