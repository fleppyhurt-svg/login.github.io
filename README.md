# login.github.io
Nazar and Roma
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <title>Рома и Назар: 3D MMA Forest</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: sans-serif; user-select: none; }
        #canvas-container { width: 100%; height: 100vh; }
        
        /* UI */
        #ui { position: absolute; top: 20px; left: 20px; color: white; font-size: 24px; text-shadow: 2px 2px 2px black; pointer-events: none; z-index: 10; }
        #msg { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); color: yellow; font-size: 40px; text-align: center; display: none; text-shadow: 4px 4px 4px black; z-index: 11; }

        /* Джойстики */
        .joystick-zone { position: absolute; bottom: 30px; width: 40vw; height: 40vw; max-width: 200px; max-height: 200px; z-index: 20; opacity: 0.7; }
        #zone_roma { left: 30px; }
        #zone_nazar { right: 30px; }
    </style>
</head>
<body>

<div id="ui">🍄 Грибы: <span id="score">0</span> / 10</div>
<div id="msg"></div>
<div id="canvas-container"></div>

<div id="zone_roma" class="joystick-zone"></div>
<div id="zone_nazar" class="joystick-zone"></div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/nipplejs/0.9.0/nipplejs.min.js"></script>

<script>
    // --- НАСТРОЙКИ СЦЕНЫ ---
    let scene, camera, renderer, clock;
    let ground;
    const items = [];
    const monsters = [];
    let mushroomsCollected = 0;
    let gameState = 'playing'; // 'playing', 'bossfight', 'win'

    // --- ИГРОВЫЕ ОБЪЕКТЫ ---
    const players = {
        roma: { mesh: null, mix: new THREE.Vector2(), speed: 0.3, hp: 100, color: 0x3498db },
        nazar: { mesh: null, mix: new THREE.Vector2(), speed: 0.3, hp: 100, color: 0xe74c3c }
    };
    let boss = { mesh: null, active: false, hp: 1000, speed: 0.1, color: 0x2c3e50 };

    init();
    animate();

    function init() {
        // 1. Scene & Clock
        scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87ceeb); // Небо
        scene.fog = new THREE.Fog(0x87ceeb, 20, 100);
        clock = new THREE.Clock();

        // 2. Camera
        camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.set(0, 15, 25);
        camera.lookAt(0, 0, 0);

        // 3. Renderer
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.shadowMap.enabled = true;
        document.getElementById('canvas-container').appendChild(renderer.domElement);

        // 4. Lights
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
        scene.add(ambientLight);
        const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
        directionalLight.position.set(10, 20, 10);
        directionalLight.castShadow = true;
        scene.add(directionalLight);

        // 5. Ground (Лес)
        const groundGeometry = new THREE.PlaneGeometry(100, 100);
        const groundMaterial = new THREE.MeshPhongMaterial({ color: 0x2d5a27 });
        ground = new THREE.Mesh(groundGeometry, groundMaterial);
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        scene.add(ground);

        // 6. Создание персонажей (Кубы для простоты, но в 3D)
        players.roma.mesh = createPlayerMesh(players.roma.color, -5);
        players.nazar.mesh = createPlayerMesh(players.nazar.color, 5);

        // 7. Подготовка босса (скрыт)
        createBossMesh();

        // 8. Mobile Controls (Джойстики)
        setupJoysticks();

        // 9. Spawners
        setInterval(spawnMushroom, 3000);
        setInterval(spawnMonster, 4000);

        window.addEventListener('resize', onWindowResize, false);
    }

    // --- ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ 3D ---

    function createPlayerMesh(color, xOffset) {
        // Тело
        const geometry = new THREE.BoxGeometry(1, 2, 1);
        const material = new THREE.MeshPhongMaterial({ color: color });
        const mesh = new THREE.Mesh(geometry, material);
        mesh.position.set(xOffset, 1, 0);
        mesh.castShadow = true;
        scene.add(mesh);
        // "Глаза" чтобы понимать куда смотрит
        const eyeGeo = new THREE.BoxGeometry(0.2, 0.2, 0.2);
        const eyeMat = new THREE.MeshPhongMaterial({color: 0x000000});
        const eye = new THREE.Mesh(eyeGeo, eyeMat);
        eye.position.set(0.3, 0.5, 0.5);
        mesh.add(eye);
        const eye2 = eye.clone(); eye2.position.x = -0.3;
        mesh.add(eye2);

        return mesh;
    }

    function createBossMesh() {
        const geometry = new THREE.BoxGeometry(4, 5, 2);
        const material = new THREE.MeshPhongMaterial({ color: boss.color });
        boss.mesh = new THREE.Mesh(geometry, material);
        boss.mesh.position.set(0, 2.5, -30); // Далеко
        boss.mesh.castShadow = true;
        boss.mesh.visible = false;
        scene.add(boss.mesh);
        
        // Надпись MELL (очень упрощенно, просто блок)
        const crownGeo = new THREE.TorusGeometry( 1, 0.2, 8, 6 );
        const crownMat = new THREE.MeshPhongMaterial( { color: 0xffd700 } );
        const crown = new THREE.Mesh( crownGeo, crownMat );
        crown.position.y = 3.5;
        crown.rotation.x = Math.PI/2;
        boss.mesh.add(crown);
    }

    function spawnMushroom() {
        if (gameState !== 'playing' || items.length > 5) return;
        // Конус как гриб
        const geometry = new THREE.ConeGeometry(0.3, 0.7, 8);
        const material = new THREE.MeshPhongMaterial({ color: 0xffd700 });
        const mesh = new THREE.Mesh(geometry, material);
        mesh.position.set(Math.random() * 40 - 20, 0.35, Math.random() * 40 - 20);
        mesh.castShadow = true;
        scene.add(mesh);
        items.push(mesh);
    }

    function spawnMonster() {
        if (gameState !== 'playing' || monsters.length > 3) return;
        // Шар как монстр
        const geometry = new THREE.SphereGeometry(0.7, 16, 16);
        const material = new THREE.MeshPhongMaterial({ color: 0x8e44ad });
        const mesh = new THREE.Mesh(geometry, material);
        // Спавн спереди по Z
        mesh.position.set(Math.random() * 40 - 20, 0.7, -40);
        mesh.castShadow = true;
        scene.add(mesh);
        monsters.push(mesh);
    }

    // --- УПРАВЛЕНИЕ (Nipple.js) ---

    function setupJoysticks() {
        // Рома (Слева)
        const managerRoma = nipplejs.create({
            zone: document.getElementById('zone_roma'),
            mode: 'static',
            position: { left: '50%', top: '50%' },
            color: 'blue'
        });
        managerRoma.on('move', (evt, data) => {
            if (data.vector) {
                players.roma.mix.x = data.vector.x;
                players.roma.mix.y = data.vector.y;
            }
        });
        managerRoma.on('end', () => { players.roma.mix.set(0, 0); });

        // Назар (Справа)
        const managerNazar = nipplejs.create({
            zone: document.getElementById('zone_nazar'),
            mode: 'static',
            position: { left: '50%', top: '50%' },
            color: 'red'
        });
        managerNazar.on('move', (evt, data) => {
            if (data.vector) {
                players.nazar.mix.x = data.vector.x;
                players.nazar.mix.y = data.vector.y;
            }
        });
        managerNazar.on('end', () => { players.nazar.mix.set(0, 0); });
    }

    // --- ИГРОВОЙ ЦИКЛ (ЛОГИКА) ---

    function animate() {
        requestAnimationFrame(animate);
        const delta = clock.getDelta();

        if (gameState === 'win') {
             renderer.render(scene, camera);
             return;
        }

        // 1. Движение игроков
        movePlayer(players.roma);
        movePlayer(players.nazar);

        // 2. Сбор грибов
        checkMushroomCollision();

        // 3. Логика монстров
        updateMonsters(delta);

        // 4. Логика босса
        updateBoss(delta);

        // 5. Камера следит за игроками (средняя точка)
        const avgX = (players.roma.mesh.position.x + players.nazar.mesh.position.x) / 2;
        const avgZ = (players.roma.mesh.position.z + players.nazar.mesh.position.z) / 2;
        camera.position.x = THREE.MathUtils.lerp(camera.position.x, avgX, 0.05);
        camera.position.z = THREE.MathUtils.lerp(camera.position.z, avgZ + 25, 0.05);
        camera.lookAt(avgX, 0, avgZ);

        renderer.render(scene, camera);
    }

    function movePlayer(player) {
        if (!player.mesh) return;
        
        const moveVec = new THREE.Vector3(player.mix.x, 0, -player.mix.y);
        moveVec.normalize().multiplyScalar(player.speed);
        
        // Границы мира
        const nextPos = player.mesh.position.clone().add(moveVec);
        if (Math.abs(nextPos.x) < 45 && Math.abs(nextPos.z) < 45) {
            player.mesh.position.add(moveVec);
        }

        // Поворот в сторону движения
        if (moveVec.lengthSq() > 0) {
            const targetRotation = Math.atan2(moveVec.x, moveVec.z);
            player.mesh.rotation.y = THREE.MathUtils.lerp(player.mesh.rotation.y, targetRotation, 0.2);
        }
    }

    function checkMushroomCollision() {
        if (gameState !== 'playing') return;

        [players.roma.mesh, players.nazar.mesh].forEach(playerMesh => {
            for (let i = items.length - 1; i >= 0; i--) {
                const mush = items[i];
                if (playerMesh.position.distanceTo(mush.position) < 1.5) {
                    // Собрал
                    scene.remove(mush);
                    items.splice(i, 1);
                    mushroomsCollected++;
                    document.getElementById('score').innerText = mushroomsCollected;

                    // Text effect (в 3D сложнее, пропустим для простоты)

                    if (mushroomsCollected >= 10) {
                        startBossFight();
                    }
                }
            }
        });
    }

    function updateMonsters(delta) {
        if (gameState !== 'playing') return;

        for (let i = monsters.length - 1; i >= 0; i--) {
            const mon = monsters[i];
            
            // Идут на игроков
            const target = (i % 2 === 0) ? players.roma.mesh : players.nazar.mesh;
            const dir = new THREE.Vector3().subVectors(target.position, mon.position).normalize();
            mon.position.add(dir.multiplyScalar(0.05));

            // Проверка ММА боя
            [players.roma.mesh, players.nazar.mesh].forEach(pMesh => {
                if (mon.position.distanceTo(pMesh.position) < 1.8) {
                    // Победили монстра (ММА)
                    scene.remove(mon);
                    monsters.splice(i, 1);
                    // Эффект удара (визуальный плейсхолдер)
                    pMesh.scale.set(1.2, 1.2, 1.2); 
                    setTimeout(()=> pMesh.scale.set(1,1,1), 100);
                }
            });

            // Если монстр улетел далеко
            if (mon.position.z > 50) {
                scene.remove(mon);
                monsters.splice(i, 1);
            }
        }
    }

    function startBossFight() {
        gameState = 'bossfight';
        
        // Удалить оставшихся монстров
        monsters.forEach(m => scene.remove(m));
        monsters.length = 0;

        const msg = document.getElementById('msg');
        msg.innerHTML = "MELLSTROY &<br>DRAGON MONEY BAND";
        msg.style.display = 'block';
        
        boss.mesh.visible = true;
        boss.active = true;

        setTimeout(() => msg.style.display = 'none', 4000);
    }

    function updateBoss(delta) {
        if (!boss.active) return;

        // Босс медленно идет к центру или к игрокам
        const targetPos = new THREE.Vector3(0, 2.5, 0); 
        const dir = new THREE.Vector3().subVectors(targetPos, boss.mesh.position).normalize();
        boss.mesh.position.add(dir.multiplyScalar(boss.speed));

        // Игроки бьют босса
        let playerNear = false;
        [players.roma.mesh, players.nazar.mesh].forEach(pMesh => {
            if (pMesh.position.distanceTo(boss.mesh.position) < 4) {
                boss.hp -= 2; // Урон боссу
                playerNear = true;
            }
        });

        if (playerNear) {
             boss.mesh.material.color.set(0xffffff); // Мигание при уроне
        } else {
             boss.mesh.material.color.set(boss.color);
        }

        if (boss.hp <= 0) {
            win();
        }
    }

    function win() {
        gameState = 'win';
        boss.active = false;
        scene.remove(boss.mesh);
        
        const msg = document.getElementById('msg');
        msg.innerHTML = "ПОБЕДА!<br>Рома и Назар отстояли лес!";
        msg.style.display = 'block';
    }

    function onWindowResize() {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    }

</script>
</body>
</html>
