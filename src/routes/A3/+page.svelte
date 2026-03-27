<script lang="ts">
    import { onMount } from "svelte";
    import * as THREE from "three";
    import { FontLoader, Font } from "three/addons/loaders/FontLoader.js";
    import { TextGeometry } from "three/addons/geometries/TextGeometry.js";
    import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
    import type { TMovie } from "../../types";
    import { addGround, onWindowResize, loadModels } from "$lib/Helper-3D";

    import * as d3 from "d3";

    let movies: TMovie[] = $state([]);

    // Stacked data: each year has counts per genre
    type TStackRow = {
        year: string;
        Comedy: number;
        Romance: number;
        Drama: number;
    };

    let stackData: TStackRow[] = $state([]);
    const genres = ["Comedy", "Romance", "Drama"];
    const genreColors: Record<string, number> = {
        Comedy: 0xf4a261,  // warm orange
        Romance: 0xe76f51,  // coral red
        Drama: 0x2a9d8f,   // teal
    };

    function normalizeModel(sceneGroup: THREE.Group, targetSize: number) {
        // 1. Calculate the current size of the model
        const box = new THREE.Box3().setFromObject(sceneGroup);
        const size = box.getSize(new THREE.Vector3());
        
        // 2. Find the largest dimension (width, height, or depth)
        const maxDim = Math.max(size.x, size.y, size.z);
        if (maxDim === 0) return sceneGroup; // Avoid division by zero

        // 3. Scale the model so its largest side equals targetSize
        const scaleFactor = targetSize / maxDim;
        sceneGroup.scale.setScalar(scaleFactor);

        // 4. Center the model (optional but highly recommended for rotations)
        const center = box.getCenter(new THREE.Vector3());
        sceneGroup.position.x -= center.x * scaleFactor;
        sceneGroup.position.y -= center.y * scaleFactor;
        sceneGroup.position.z -= center.z * scaleFactor;

        return sceneGroup;
}

    // === TODO-1: Load Data ===
    // Currently using hardcoded dummy data.
    // TODO: Load the summer_movies.csv, then aggregate
    //   the real movie data into stackData.
    //   - Group movies by year (movie.year.getFullYear())
    //   - Count how many movies belong to each genre (Comedy, Romance, Drama)
    //   - Build a TStackRow[] array sorted by year
    async function loadCsv() {
        // 1. Load the data
    const rawData = await d3.csv("summer_movies.csv");

    const yearMap = new Map();
    const targetGenres = ["Comedy", "Romance", "Drama"];

    rawData.forEach(d => {
        // 2. Clean the Year (handles "1946.0" or "1946")
        if (!d.year) return; // Skip rows with missing years
        const yearStr = Math.floor(+d.year).toString();

        // 3. Initialize the row if it's the first time we see this year
        if (!yearMap.has(yearStr)) {
            yearMap.set(yearStr, { year: yearStr, Comedy: 0, Romance: 0, Drama: 0});
        }

        const row = yearMap.get(yearStr);
        
        // 4. Handle multiple genres in one string (e.g., "Comedy,Drama")
        if (d.genres) {
            const movieGenres = d.genres.split(',').map(g => g.trim());
            
            targetGenres.forEach(genre => {
                if (movieGenres.includes(genre)) {
                    row[genre]++;
                }
            });
        }
    });

    // 5. Convert to array and sort chronologically
    stackData = Array.from(yearMap.values())
        .sort((a, b) => (+a.year) - (+b.year));

    console.log("Real Movie Stack Data:", stackData);
}

    let container: HTMLElement;
    let camera: THREE.PerspectiveCamera;
    let scene: THREE.Scene;
    let renderer: THREE.WebGLRenderer;
    const FLOOR = -250;

    const morphs: Array<THREE.Mesh> = [];
    let mixer: THREE.AnimationMixer;
    const clock = new THREE.Clock();

    const horseMorphs: Array<THREE.Mesh> = [];
    let horseMixer: THREE.AnimationMixer;
    const ORBIT_RX = 850;
    const ORBIT_RZ = 400;


    let animating = $state(false);

    onMount(async () => {
        await loadCsv();
        init(window.innerWidth, window.innerHeight);
    });
    const hoppingModels: THREE.Group[] = [];

    function init(SCREEN_WIDTH: number, SCREEN_HEIGHT: number) {
        // Renderer
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.setSize(SCREEN_WIDTH, SCREEN_HEIGHT);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFShadowMap;
        container.appendChild(renderer.domElement);

        // Camera
        camera = new THREE.PerspectiveCamera(
            23,
            SCREEN_WIDTH / SCREEN_HEIGHT,
            10,
            5000,
        );
        camera.position.set(0, 100, 2800);

        // Scene
        scene = new THREE.Scene();
        new THREE.TextureLoader().load("3d/sky.jpg", (texture) => {
            texture.repeat.set(0.8, 1);
            scene.background = texture;
        });

        // Ambient Light
        const ambient = new THREE.AmbientLight(0xffffff);
        scene.add(ambient);

        // Directional Light
        const light = new THREE.DirectionalLight(0xffffff, 3);
        light.position.set(0, 1500, 1000);
        light.castShadow = true;
        Object.assign(light.shadow.camera, {
            top: 2000,
            bottom: -2000,
            left: -2000,
            right: 2000,
            near: 1200,
            far: 2500,
        });
        light.shadow.bias = 0.0001;
        light.shadow.mapSize.width = 2048;
        light.shadow.mapSize.height = 1024;
        scene.add(light);

        // Ground
        addGround(scene, FLOOR, "3d/grasslight-big.jpg");

        // Title + Stacked Bars
        const fontLoader = new FontLoader();
        fontLoader.load("3d/helvetiker_bold.typeface.json", (font: Font) => {
            // Title
            const textGeo = new TextGeometry("summer movies", {
                font: font,
                size: 30,
                depth: 10,
            });
            textGeo.computeBoundingBox();
            const centerOffset =
                -0.5 *
                (textGeo!.boundingBox!.max.x - textGeo!.boundingBox!.min.x);
            const textMaterial = new THREE.MeshStandardMaterial({
                color: 0x449900,
            });
            const titleMesh = new THREE.Mesh(textGeo, textMaterial);
            titleMesh.position.x = centerOffset;
            titleMesh.position.y = FLOOR + 500;
            titleMesh.castShadow = true;
            scene.add(titleMesh);

            createUnits(scene, font);
            createLegend(scene, font);
        });

        // === TODO-2: Add Expressive 3D Objects ===
        // Make the scene more expressive, engaging and fun by changing or adding
        // at least one 3D object! You do NOT need to replace all existing objects —
        // modifying or adding just one is sufficient.
        // Animation is required (e.g., rotation, bobbing, or using a model's built-in animation clips).
        //
        // Where to find free GLB models:
        //   - https://poly.pizza (search, download as .glb)
        //   - https://sketchfab.com (filter by "Downloadable" + "glTF")
        //   - https://kenney.nl/assets (game-style assets, some 3D)
        //   - https://www.fab.com/search?q=3d+object&asset_formats=glb&asset_formats=converted-files&is_free=1
        //
        // Tips:
        //   - Place your .glb files in the /static/3d/ folder
        //   - Adjust the `scale` value if your model is too big or small
        //   - If a model has no built-in animations, add a custom one
        //     (e.g., rotate or float the mesh in the animate() function)
        //   - If a model has no animations, guard the animation code:
        //     if (gltf.animations.length > 0) { ... }
        //   - Feel free to add extra static models as decoration (trees, buildings, etc.)!

        // Load bird models (linear motion)
        const linearAssets = [
            {
                path: "3d/Flamingo.glb",
                speed: 350,
                y: FLOOR + 550,
                targetSize: 150, // Use this with the helper
                // Built-in animation logic
                update: (mesh: THREE.Object3D, delta: number) => {
                    // Flamingo uses mixer, so we leave this empty or add a slight tilt
                }
            }
        ];
        /*const birdModels = [
            {
                path: "3d/Flamingo.glb", // TODO: replace with your model
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: 100,
                scale: 2,
            },
            {
                path: "3d/Dice.glb", // TODO: replace with your model
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: -100,
                scale: 0.1,
            },
            {
                path: "3d/paintKitMini.glb", // TODO: replace with your model
                speed: 350,
                duration: 0.5,
                x: 500 - Math.random() * 500,
                y: FLOOR + 500,
                z: 700,
                scale: 110.0,
            },
        ];*/
        mixer = loadModels(linearAssets, scene, mixer, morphs);
console.log(linearAssets)
        // Load horse models (orbit around data)
        horseMixer = new THREE.AnimationMixer(scene);
        console.log(horseMixer)
        const gltfLoader = new GLTFLoader();
        const horseConfigs = [
            { startAngle: 0, orbitSpeed: 0.4 },
            { startAngle: Math.PI, orbitSpeed: 0.4 },
        ];
        const customModels = [
            { path: "3d/Dice.glb", targetSize: 80, y: FLOOR + 400, speed: 200},
            { path: "3d/Camera.glb", targetSize: 120, y: FLOOR + 200, speed: 100}
        ];

        customModels.forEach(config => {
            const loader = new GLTFLoader();
            loader.load(config.path, (gltf) => {
                // Use your normalization helper to fix the "invisible/huge" issue [cite: 8]
                const model = normalizeModel(gltf.scene, config.targetSize);
                
                model.position.set(-900, config.y, 0); // Start off-screen
                model.userData.speed = config.speed;
                
                scene.add(model);
                morphs.push(model as any); // Add to your animation loop 
            });
        });
        const hoppingAssets = [
                {
                    path: "3d/penguin.glb", // Replace with your file
                    targetSize: 100,
                    x: 500,
                    z: 200,
                    hopHeight: 50,  // How high it jumps
                    hopSpeed: 4     // How fast it jumps
                }
        ];
        
        horseConfigs.forEach((config) => {
            gltfLoader.load("3d/Dice.glb", (gltf) => {
                // Search the whole scene for the first mesh
                let mesh: THREE.Mesh | undefined;
                gltf.scene.traverse((child) => {
                    if (child instanceof THREE.Mesh && !mesh) {
                        console.log(child)
                        mesh = child.clone() as THREE.Mesh;
                        console.log(mesh)
                    }
                });

                if (mesh) {
                    mesh.scale.set(500, 500, 500); // Try a larger start scale
                mesh.castShadow = true;

                mesh.userData.angle = config.startAngle;
                mesh.userData.orbitSpeed = config.orbitSpeed;
                mesh.position.set(
                    ORBIT_RX * Math.cos(config.startAngle),
                    FLOOR,
                    ORBIT_RZ * Math.sin(config.startAngle),
                );

                scene.add(mesh);
                horseMorphs.push(mesh);
                console.log("whatever you like")
                }
            });
        });

        hoppingAssets.forEach(config => {
            const gltfLoader = new GLTFLoader();
            gltfLoader.load(config.path, (gltf) => {
                // Normalize handles the targetSize/maxDim calculation for you [cite: 8, 11]
                const model = normalizeModel(gltf.scene, config.targetSize);
                
                model.position.set(config.x, FLOOR, config.z); //[cite: 70]
                
                // Save the data to the model itself!
                model.userData = { 
                    baseY: FLOOR, 
                    hopHeight: config.hopHeight, 
                    hopSpeed: config.hopSpeed,
                    baseScaleY: model.scale.y // This "captures" the scale for later [cite: 11]
                };
                
                scene.add(model);
                hoppingModels.push(model); //[cite: 72]
            });
        });
        // === END TODO-2 ===

        // Handle resize
        window.addEventListener("resize", () =>
            onWindowResize(
                camera,
                renderer,
                window.innerWidth,
                window.innerHeight,
            ),
        );

        renderer.setAnimationLoop(animate);
    }

    // === TODO-3: Unit Visualization ===
    //
    // TODO-3a: Define a unique 3D shape and material per genre.
    //   Docs: https://threejs.org/docs/#api/en/geometries
    //   Docs: https://threejs.org/docs/#api/en/materials/MeshPhysicalMaterial
    
    function createGenreGeometry(genre: string): THREE.BufferGeometry {
        switch (genre) {
        case "Comedy":
            return new THREE.TorusGeometry(5, 2, 8, 16); // A "donut" shape
        case "Romance":
            return new THREE.SphereGeometry(6, 16, 16);  // A smooth sphere
        case "Drama":
            return new THREE.ConeGeometry(6, 12, 4);     // A pyramid/cone
        default:
            return new THREE.BoxGeometry(10, 10, 10);
        }
    }
    //     // TODO: Return a different geometry for each genre,
    //   see the documentation: https://threejs.org/docs/#api/en/geometries
    //     //   Comedy :
    //     //   Romance :
    //     //   Drama   :
    // }
    //
    function createGenreMaterial(genre: string): THREE.Material {
        return new THREE.MeshPhysicalMaterial({
            color: genreColors[genre],
            metalness: 0.2,
            roughness: 0.1,
            transmission: 0.5, // Adds a slightly "glassy" look
            thickness: 0.5
        });
    }
    //     // TODO: Return a material using genreColors[genre], 
    //   see the documentation: https://threejs.org/docs/#api/en/materials/MeshPhysicalMaterial
    // }

    // TODO-3b: Implement createUnits(), then call it instead of createBars()
    //   Change line 141:  createBars(scene, font);
    //   To:               createUnits(scene, font);
    //
    function createUnits(scene: THREE.Scene, font: Font) {
        if (stackData.length === 0) return;

        const barMaxWidth = Math.min(stackData.length * 80, 1400);
        const unitSize = 12; // Vertical space each object takes up

        const xScale = d3.scaleBand()
            .domain(stackData.map((d) => d.year))
            .range([-barMaxWidth / 2, barMaxWidth / 2])
            .padding(0.1);

        const bandwidth = xScale.bandwidth();

        stackData.forEach((row) => {
            const xPos = xScale(row.year)! + bandwidth / 2;
            let currentY = FLOOR;

            genres.forEach((genre) => {
                const count = row[genre as keyof TStackRow] as number;
                
                for (let i = 0; i < count; i++) {
                    const geo = createGenreGeometry(genre);
                    const mat = createGenreMaterial(genre);
                    const mesh = new THREE.Mesh(geo, mat);

                    // Stack each unit vertically
                    mesh.position.set(xPos, currentY + unitSize / 2, 0);
                    mesh.castShadow = true;
                    mesh.receiveShadow = true;
                    scene.add(mesh);

                    currentY += unitSize;
                }
            });

            // Add Year Labels (reused from your createBars logic)
            const textGeo = new TextGeometry(row.year, { font, size: 6, depth: 2 });
            textGeo.computeBoundingBox();
            const textWidth = textGeo.boundingBox!.max.x - textGeo.boundingBox!.min.x;
            const textMesh = new THREE.Mesh(textGeo, new THREE.MeshPhysicalMaterial({ color: 0x000000 }));
            textMesh.position.set(xPos - textWidth / 2, FLOOR + 5, 15);
            scene.add(textMesh);
        });
    }
    //     if (stackData.length === 0) return;
    //
    //     const barMaxWidth = Math.min(stackData.length * 80, 1400);
    //     const unitSize = 10; // height of each unit object
    //
    //     const xScale = d3
    //         .scaleBand()
    //         .domain(stackData.map((d) => d.year))
    //         .range([-barMaxWidth / 2, barMaxWidth / 2])
    //         .padding(0.1);
    //
    //     const bandwidth = xScale.bandwidth();
    //
    //     // Stack unit objects per year
    //     stackData.forEach((row) => {
    //         const xPos = xScale(row.year)! + bandwidth / 2;
    //         let currentY = FLOOR;
    //
    //         genres.forEach((genre) => {
    //             const count = row[genre as keyof TStackRow] as number;
    //             for (let i = 0; i < count; i++) {
    //                 const geo = // TODO: use createGenreGeometry(genre)
    //                 const mat = // TODO: use createGenreMaterial(genre)
    //                 const mesh = new THREE.Mesh(geo, mat);
    //                 // TODO: set mesh position at (xPos, currentY + unitSize / 2, 0)
    //                 // TODO: enable castShadow and receiveShadow
    //                 // TODO: add mesh to scene
    //                 // TODO: advance currentY by unitSize
    //             }
    //         });
    //     });
    //
    //     // Year labels — you can reuse the year label code from createBars() below
    // }
    //
    // TODO-3c: Update createLegend() to use genre shapes and materials.

    function createBars(scene: THREE.Scene, font: Font) {
        if (stackData.length === 0) return;

        const barMaxWidth = Math.min(stackData.length * 80, 1400);

        // D3 scales
        const xScale = d3
            .scaleBand()
            .domain(stackData.map((d) => d.year))
            .range([-barMaxWidth / 2, barMaxWidth / 2])
            .padding(0.1);

        const bandwidth = xScale.bandwidth();
        const barWidth = Math.min(bandwidth * 0.7, 25);

        // One colored bar segment per genre, stacked per year
        stackData.forEach((row) => {
            const xPos = xScale(row.year)! + bandwidth / 2;
            let currentY = FLOOR;

            genres.forEach((genre) => {
                const count = row[genre as keyof TStackRow] as number;
                const barHeight = count * 10;
                if (barHeight === 0) return;

                const geometry = new THREE.BoxGeometry(barWidth, barHeight, barWidth);
                const material = new THREE.MeshPhysicalMaterial({ color: genreColors[genre] });
                const mesh = new THREE.Mesh(geometry, material);
                mesh.position.set(xPos, currentY + barHeight / 2, 0);
                mesh.castShadow = true;
                mesh.receiveShadow = true;
                scene.add(mesh);
                currentY += barHeight;
            });
        });

        // Year labels
        stackData.forEach((row) => {
            const x = xScale(row.year)! + bandwidth / 2;
            const textGeo = new TextGeometry(row.year, {
                font: font,
                size: 6,
                depth: 2,
            });
            textGeo.computeBoundingBox();
            const textWidth = textGeo.boundingBox!.max.x - textGeo.boundingBox!.min.x;
            const textMat = new THREE.MeshPhysicalMaterial({ color: 0x000000 });
            const textMesh = new THREE.Mesh(textGeo, textMat);
            textMesh.position.set(x - textWidth / 2, FLOOR + 5, barWidth / 2 + 5);
            textMesh.castShadow = true;
            scene.add(textMesh);
        });
    }
    // Update your new Legend code here — see TODO-3c
    function createLegend(scene: THREE.Scene, font: Font) {
        const legendX = 220; // Move it slightly to the side
        const legendStartY = FLOOR + 450;
        const spacing = 50;

        genres.forEach((genre, i) => {
            const y = legendStartY - i * spacing;

            // Use the new genre-specific geometry and material
            const swatchGeo = createGenreGeometry(genre);
            const swatchMat = createGenreMaterial(genre);
            const swatch = new THREE.Mesh(swatchGeo, swatchMat);
            
            swatch.position.set(legendX, y, 0);
            swatch.scale.set(1.5, 1.5, 1.5); // Make legend items slightly larger
            swatch.castShadow = true;
            scene.add(swatch);

            const textGeo = new TextGeometry(genre, { font, size: 12, depth: 3 });
            const textMesh = new THREE.Mesh(textGeo, new THREE.MeshPhysicalMaterial({ color: 0x000000 }));
            textMesh.position.set(legendX + 25, y - 5, 0);
            scene.add(textMesh);
        });
    }
        /*
        const legendX = 180;
        const legendStartY = FLOOR + 420;
        const spacing = 40;
        const swatchSize = 15;

        genres.forEach((genre, i) => {
            const y = legendStartY - i * spacing;

            const swatchGeo = new THREE.BoxGeometry(swatchSize, swatchSize, swatchSize);
            const swatchMat = new THREE.MeshPhysicalMaterial({ color: genreColors[genre] });
            const swatch = new THREE.Mesh(swatchGeo, swatchMat);
            swatch.position.set(legendX, y, 0);
            swatch.castShadow = true;
            scene.add(swatch);

            const textGeo = new TextGeometry(genre, {
                font: font,
                size: 10,
                depth: 3,
            });
            const textMat = new THREE.MeshPhysicalMaterial({ color: 0x000000 });
            const textMesh = new THREE.Mesh(textGeo, textMat);
            textMesh.position.set(legendX + 18, y - 5, 0);
            textMesh.castShadow = true;
            scene.add(textMesh);
        });
    }*/

    function animate() {
        const delta = clock.getDelta(); //[cite: 103]
        const elapsed = clock.getElapsedTime(); //[cite: 104]

        if (animating) { //[cite: 104]
            mixer.update(delta); //[cite: 104]

            // Fix the Hopping Logic: No more ReferenceErrors!
            hoppingModels.forEach((model) => {
                const { baseY, hopHeight, hopSpeed, baseScaleY } = model.userData;
                
                // Jump logic
                model.position.y = baseY + Math.abs(Math.sin(elapsed * hopSpeed)) * hopHeight;
                
                // Squish logic using the stored baseScaleY 
                const scaleEffect = 1 + Math.sin(elapsed * hopSpeed) * 0.7;
                model.scale.y = baseScaleY * scaleEffect; 
            });

            // Keep your Dice orbiting (the horseMorphs) [cite: 63-68]
            horseMorphs.forEach((mesh) => {
                mesh.userData.angle += mesh.userData.orbitSpeed * delta;
                mesh.position.x = ORBIT_RX * Math.cos(mesh.userData.angle);
                mesh.position.z = ORBIT_RZ * Math.sin(mesh.userData.angle);
                mesh.rotation.y += delta;
            });

            // Keep your Linear models (the morphs) [cite: 107]
            morphs.forEach((mesh) => {
                mesh.position.x += (mesh.userData.speed || 0) * delta;
                mesh.rotation.y += delta * 2; //[cite: 107]
                
                if (mesh.position.x > 1500) mesh.position.x = -1500; //[cite: 108]
            });
        }
        renderer.render(scene, camera); //[cite: 109]
    }
</script>

<div bind:this={container} class="container"></div>

<button class="toggle-btn" onclick={() => animating = !animating}>
    {animating ? "Pause" : "Play"} Animation
</button>

<style>
    div.container {
        width: 100%;
        height: 100%;
    }

    .toggle-btn {
        position: fixed;
        bottom: 10px;
        right: 10px;
        padding: 8px 10px;
        font-size: 12px;
        border: none;
        border-radius: 20px;
        background: rgba(0, 0, 0, 0.30);
        color: white;
        cursor: pointer;
        z-index: 10;
    }

    .toggle-btn:hover {
        background: rgba(0, 0, 0, 0.8);
    }
</style>
  