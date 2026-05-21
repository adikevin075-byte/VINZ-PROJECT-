<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Game Beam Lighting 3D - Dark Seeker</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #000; font-family: Arial, sans-serif; }
        #ui {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            color: white;
            text-align: center;
            pointer-events: none;
        }
        #crosshair {
            width: 20px;
            height: 20px;
            border: 2px solid rgba(255, 255, 255, 0.5);
            border-radius: 50%;
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            pointer-events: none;
        }
        #score {
            position: absolute;
            top: 20px;
            left: 20px;
            color: white;
            font-size: 20px;
        }
        h1 { margin: 0; font-size: 40px; text-shadow: 0 0 10px #fff; }
        p { font-size: 18px; color: #aaa; }
    </style>
</head>
<body>

    <div id="score">Score: 0</div>
    <div id="crosshair"></div>
    
    <div id="ui">
        <h1>DARK SEEKER</h1>
        <p>Klik layar untuk memulai</p>
        <p>Gerak: W A S D | Blick: Mouse | Senter: Otomatis</p>
    </div>

    <!-- Import Three.js sebagai Module -->
    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
                "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
            }
        }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';

        // 1. SETUP SCENE
        const scene = new THREE.Scene();
        scene.fog = new THREE.FogExp2(0x000000, 0.03); // Fog gelap agar senter memantul bagus

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.y = 1.6; // Tinggi mata manusia

        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.shadowMap.enabled = true;
        document.body.appendChild(renderer.domElement);

        // 2. LIGHTING (THE CORE MECHANIC)
        // Cahaya redup ambient agar ruangan tidak totally hitam tapi tetap gelap
        const ambientLight = new THREE.AmbientLight(0x111111); 
        scene.add(ambientLight);

        // THE BEAM (SENTER)
        const spotLight = new THREE.SpotLight(0xffffff, 200); // Warna putih, intensitas 200
        spotLight.angle = Math.PI / 6;
        spotLight.penumbra = 0.5; // Batas kabur cahaya
        spotLight.decay = 1.5;
        spotLight.distance = 50;
        spotLight.castShadow = true;
        
        // Posisikan senter relative terhadap kamera
        spotLight.position.set(0.5, -0.5, 0); 
        camera.add(spotLight); // Senter mengikuti kamera
        spotLight.target = camera; // Arahnya sama dengan hadap kamera
        
        // Agar senter arahnya selalu向前
        const targetObject = new THREE.Object3D();
        camera.add(targetObject);
        spotLight.target = targetObject;
        
        scene.add(camera);

        // 3. WORLD OBJECTS
        // Lantai
        const floorGeometry = new THREE.PlaneGeometry(100, 100);
        const floorMaterial = new THREE.MeshStandardMaterial({ color: 0x222222, roughness: 0.8 });
        const floor = new THREE.Mesh(floorGeometry, floorMaterial);
        floor.rotation.x = -Math.PI / 2;
        floor.receiveShadow = true;
        scene.add(floor);

        // Targets (Kubus Musuh)
        const targets = [];
        const boxGeo = new THREE.BoxGeometry(1, 1, 1);
        
        function createTarget() {
            // Gunakan MeshStandardMaterial agar bereaksi terhadap cahaya
            const material = new THREE.MeshStandardMaterial({ 
                color: 0x000000, // Hitam sempurna di gelap
                roughness: 0.1 
            });
            const cube = new THREE.Mesh(boxGeo, material);
            
            // Posisi acak
            cube.position.x = (Math.random() - 0.5) * 40;
            cube.position.z = (Math.random() - 0.5) * 40;
            cube.position.y = 1 + Math.random() * 2;
            
            cube.castShadow = true;
            cube.receiveShadow = true;
            
            // Random rotation speed
            cube.userData = {
                rotSpeed: Math.random() * 0.05 + 0.01,
                isLit: false
            };
            
            scene.add(cube);
            targets.push(cube);
        }

        // Buat 20 target awal
        for(let i=0; i<20; i++) createTarget();

        // 4. CONTROLS & INTERACTION
        const controls = new PointerLockControls(camera, document.body);
        const uiElement = document.getElementById('ui');
        const scoreElement = document.getElementById('score');
        
        let score = 0;

        document.addEventListener('click', () => {
            controls.lock();
        });

        controls.addEventListener('lock', () => {
            uiElement.style.display = 'none';
        });

        controls.addEventListener('unlock', () => {
            uiElement.style.display = 'block';
        });

        // Movement Keys
        const moveState = { w: false, a: false, s: false, d: false };
        document.addEventListener('keydown', (e) => {
            if (e.code === 'KeyW') moveState.w = true;
            if (e.code === 'KeyA') moveState.a = true;
            if (e.code === 'KeyS') moveState.s = true;
            if (e.code === 'KeyD') moveState.d = true;
        });
        document.addEventListener('keyup', (e) => {
            if (e.code === 'KeyW') moveState.w = false;
            if (e.code === 'KeyA') moveState.a = false;
            if (e.code === 'KeyS') moveState.s = false;
            if (e.code === 'KeyD') moveState.d = false;
        });

        // Raycaster untuk mendeteksi apakah senter mengenai objek
        const raycaster = new THREE.Raycaster();
        
        document.addEventListener('mousedown', () => {
            if (controls.isLocked) {
                // Cek apa yang dilihat kamera
                raycaster.setFromCamera(new THREE.Vector2(0, 0), camera);
                const intersects = raycaster.intersectObjects(targets);

                // Efek klik: jika mengenai target, destroy itu
                if (intersects.length > 0 && intersects[0].distance < 10) { // Jarak max 10 meter
                    const hitObject = intersects[0].object;
                    
                    // Efek ledakan partikel sederhana (hilangkan objek)
                    scene.remove(hitObject);
                    const index = targets.indexOf(hitObject);
                    if (index > -1) targets.splice(index, 1);
                    
                    score += 10;
                    scoreElement.innerText = "Score: " + score;
                    
                    // Spawn target baru
                    createTarget();
                }
            }
        });

        // 5. GAME LOOP
        const clock = new THREE.Clock();

        function animate() {
            requestAnimationFrame(animate);

            const delta = clock.getDelta();

            // Gerakan Player
            if (controls.isLocked) {
                const speed = 10 * delta;
                if (moveState.w) controls.moveForward(speed);
                if (moveState.s) controls.moveForward(-speed);
                if (moveState.d) controls.moveRight(speed);
                if (moveState.a) controls.moveRight(-speed);
            }

            // Animasi Target (Berputar)
            targets.forEach(obj => {
                obj.rotation.x += obj.userData.rotSpeed;
                obj.rotation.y += obj.userData.rotSpeed;
            });

            renderer.render(scene, camera);
        }

        // Handle Window Resize
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        animate();
    </script>
</body>
</html>
