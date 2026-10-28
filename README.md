# kddco3
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Visor 3D Optimizado - Recorrido y Entorno IA</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #000;
            font-family: sans-serif;
        }
        #ui {
            position: absolute;
            top: 10px;
            left: 10px;
            background: rgba(0, 0, 0, 0.7);
            color: #fff;
            padding: 12px;
            border-radius: 8px;
            pointer-events: auto;
            z-index: 10;
        }
        button {
            background: #2563eb;
            color: white;
            border: none;
            padding: 8px 12px;
            border-radius: 4px;
            cursor: pointer;
            margin-top: 5px;
            font-weight: bold;
        }
        button:hover { background: #1d4ed8; }
        .info { font-size: 12px; color: #cbd5e1; margin-top: 4px; }
    </style>
</head>
<body>

    <div id="ui">
        <h3>Recorrido Interactivo 3D</h3>
        <button id="btnRecorrido">Iniciar Recorrido Automático</button>
        <div class="info">• Fondo y cielo procedural de bajo consumo</div>
        <div class="info">• Vehículos y personas instanciadas (GPU)</div>
    </div>

    <!-- Importar Three.js mediante ESM -->
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
        import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

        // 1. ESCENA, CÁMARA Y RENDERIZADOR (Optimizado)
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87ceeb); // Cielo base
        scene.fog = new THREE.FogExp2(0x87ceeb, 0.015);

        const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.set(0, 15, 30);

        const renderer = new THREE.WebGLRenderer({ antialias: false, powerPreference: "high-performance" });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // Limita pixeles en pantallas Retina para ahorrar recursos
        renderer.shadowMap.enabled = false; // Desactivar sombras pesadas para máximo rendimiento
        document.body.appendChild(renderer.domElement);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;

        // 2. CIELO Y FONDO COHERENTE (Procedural de bajo costo)
        const skyGeo = new THREE.SphereGeometry(500, 16, 16);
        const skyMat = new THREE.MeshBasicMaterial({ color: 0x38bdf8, side: THREE.BackSide });
        const sky = new THREE.Mesh(skyGeo, skyMat);
        scene.add(sky);

        // Luz ambiental y direccional simple (simula sol)
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.7);
        scene.add(ambientLight);
        const sunLight = new THREE.DirectionalLight(0xffffff, 0.8);
        sunLight.position.set(50, 100, 50);
        scene.add(sunLight);

        // 3. SUELO Y CALLES (Modelo base)
        const floorGeo = new THREE.PlaneGeometry(300, 300);
        const floorMat = new THREE.MeshLambertMaterial({ color: 0x1e293b });
        const floor = new THREE.Mesh(floorGeo, floorMat);
        floor.rotation.x = -Math.PI / 2;
        scene.add(floor);

        // 4. ENTORNOS, VEHÍCULOS Y PERSONAS (Instanciación masiva para gastar mínimos recursos)
        // Vehículos (Cajas simplificadas con colores)
        const carCount = 40;
        const carGeo = new THREE.BoxGeometry(2, 1, 4);
        const carMat = new THREE.MeshLambertMaterial({ color: 0xef4444 });
        const carInstanced = new THREE.InstancedMesh(carGeo, carMat, carCount);
        
        const dummy = new THREE.Object3D();
        const carData = [];
        for (let i = 0; i < carCount; i++) {
            const x = (Math.random() - 0.5) * 200;
            const z = (Math.random() - 0.5) * 200;
            carData.push({ x, z, speed: 0.2 + Math.random() * 0.4 });
            dummy.position.set(x, 0.5, z);
            dummy.updateMatrix();
            carInstanced.setMatrixAt(i, dummy.matrix);
        }
        scene.add(carInstanced);

        // Personas (Cilindros estilizados minimalistas)
        const personCount = 60;
        const personGeo = new THREE.CylinderGeometry(0.3, 0.3, 1.8, 6);
        const personMat = new THREE.MeshLambertMaterial({ color: 0x3b82f6 });
        const personInstanced = new THREE.InstancedMesh(personGeo, personMat, personCount);

        const personData = [];
        for (let i = 0; i < personCount; i++) {
            const x = (Math.random() - 0.5) * 150;
            const z = (Math.random() - 0.5) * 150;
            personData.push({ x, z, angle: Math.random() * Math.PI * 2, speed: 0.03 + Math.random() * 0.03 });
            dummy.position.set(x, 0.9, z);
            dummy.updateMatrix();
            personInstanced.setMatrixAt(i, dummy.matrix);
        }
        scene.add(personInstanced);

        // 5. SISTEMA DE RECORRIDOS (Curva basada en puntos trazados por el usuario / predefined path)
        // Puedes definir los puntos clave (waypoints) por donde quieres que pase la cámara
        const pathPoints = [
            new THREE.Vector3(0, 5, 40),
            new THREE.Vector3(30, 5, 10),
            new THREE.Vector3(20, 5, -30),
            new THREE.Vector3(-30, 5, -20),
            new THREE.Vector3(-20, 5, 20),
            new THREE.Vector3(0, 5, 40)
        ];
        const curve = new THREE.CatmullRomCurve3(pathPoints, true);
        
        let recorridoActivo = false;
        let progress = 0;

        document.getElementById('btnRecorrido').addEventListener('click', () => {
            recorridoActivo = !recorridoActivo;
            document.getElementById('btnRecorrido').innerText = recorridoActivo ? "Detener Recorrido" : "Iniciar Recorrido Automático";
        });

        // 6. BUCLE DE RENDERIZADO OPTIMIZADO (Animación de vehículos, peatones y cámara)
        const clock = new THREE.Clock();

        function animate() {
            requestAnimationFrame(animate);
            const delta = clock.getDelta();

            // Animar vehículos en bucle por las calles
            for (let i = 0; i < carCount; i++) {
                carData[i].z -= carData[i].speed;
                if (carData[i].z < -100) carData[i].z = 100;
                
                dummy.position.set(carData[i].x, 0.5, carData[i].z);
                dummy.updateMatrix();
                carInstanced.setMatrixAt(i, dummy.matrix);
            }
            carInstanced.instanceMatrix.needsUpdate = true;

            // Animar personas con movimiento aleatorio sutil
            for (let i = 0; i < personCount; i++) {
                personData[i].x += Math.cos(personData[i].angle) * personData[i].speed;
                personData[i].z += Math.sin(personData[i].angle) * personData[i].speed;
                
                if (Math.abs(personData[i].x) > 100 || Math.abs(personData[i].z) > 100) {
                    personData[i].angle += Math.PI; // Rebotar
                }

                dummy.position.set(personData[i].x, 0.9, personData[i].z);
                dummy.updateMatrix();
                personInstanced.setMatrixAt(i, dummy.matrix);
            }
            personInstanced.instanceMatrix.needsUpdate = true;

            // Control de recorrido automático guiado por la curva
            if (recorridoActivo) {
                progress += delta * 0.05; // Velocidad del recorrido
                if (progress > 1) progress = 0;

                const point = curve.getPointAt(progress);
                const tangent = curve.getTangentAt(progress);

                camera.position.copy(point);
                camera.lookAt(point.clone().add(tangent));
            } else {
                controls.update();
            }

            renderer.render(scene, camera);
        }

        animate();

        // Ajuste dinámico de ventana
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
