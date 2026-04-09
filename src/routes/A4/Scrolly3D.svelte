<script lang="ts">
    import { Scroll } from "$lib";
    import * as THREE from "three";
    import { onMount } from "svelte";
    import { onWindowResize, loadModels, addGround } from "$lib/Helper-3D";
    import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

    // define the props of this component
    type Props = {
        derivativeData: any;
        genreList: string[];
    };
    let { derivativeData, genreList }: Props = $props();
    const hoppingModels: THREE.Group[] = [];

    // Legend Penguin references
    let legendPenguinCanvas: HTMLCanvasElement;
    let legendPenguinRenderer: THREE.WebGLRenderer;
    let legendPenguinScene: THREE.Scene;
    let legendPenguinCamera: THREE.PerspectiveCamera;
    let legendPenguinModel: THREE.Group;

    let progress: number = $state(0);
    let threeJSContainer: HTMLElement;
    let camera: THREE.PerspectiveCamera; // The camera used to view the scene, typically a perspective camera for 3D rendering
    let scene: THREE.Scene; // The scene where all the objects are added
    let renderer: THREE.WebGLRenderer; // The renderer used to render the scene
    const FLOOR = -250; // The floor level of the scene

    const morphs: Array<THREE.Mesh> = []; // Array to store the morphs (animated models: horse, flamingo, parrot)
    let mixer: THREE.AnimationMixer; // The mixer used to animate the morphs

    const clock = new THREE.Clock(); // Clock to keep track of time for animations

    onMount(
        // // once the component mounts, initialize the Three.js scene
        () => {
            init(window.innerWidth * 0.6, window.innerHeight * 0.9);
        },
    );

    // Removed dynamic zooming so the camera stays at a fixed position
    $effect(() => {
        // Empty effect or just removed to disable auto-zoom effect
    });

    function init(SCREEN_WIDTH: number, SCREEN_HEIGHT: number) {
        // Renderer
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.setSize(SCREEN_WIDTH, SCREEN_HEIGHT);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFShadowMap;
        threeJSContainer.appendChild(renderer.domElement);

        // Camera
        camera = new THREE.PerspectiveCamera(
            23,
            SCREEN_WIDTH / SCREEN_HEIGHT,
            10,
            8000,
        );
        camera.position.set(0, FLOOR + 400, 3500);

        // Scene
        scene = new THREE.Scene();
        // scene.background = new THREE.Color(0x87ceeb); // Sky color (light blue)
        new THREE.TextureLoader().load("3d/sky.jpg", (texture) => {
            texture.repeat.set(0.8, 1); // repeat the texture
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

        // Add ground
        addGround(scene, FLOOR, "3d/grasslight-big.jpg");

        // Load Models
        const hoppingAssets = [
            {
                path: "3d/penguin.glb",
                targetSize: 200,
                x: 0,
                z: 50,
                hopHeight: 80,
                hopSpeed: 5,
            },
        ];
        const models: any[] = [];

        mixer = loadModels(models, scene, mixer, morphs);

        // Handle resize
        window.addEventListener("resize", () =>
            onWindowResize(
                camera,
                renderer,
                window.innerWidth / 4,
                window.innerHeight / 2,
            ),
        );

        renderer.setAnimationLoop(animate);
        const gltfLoader = new GLTFLoader();
        hoppingAssets.forEach((config) => {
            gltfLoader.load(config.path, (gltf) => {
                const model = normalizeModel(gltf.scene, config.targetSize);
                const penguinY = FLOOR + 750;
                model.position.set(config.x, penguinY, config.z);

                model.userData = {
                    baseY: penguinY,
                    hopHeight: config.hopHeight,
                    hopSpeed: config.hopSpeed,
                    baseScaleY: model.scale.y,
                };
                scene.add(model);
                hoppingModels.push(model);
            });
        });

        if (legendPenguinCanvas) {
            legendPenguinRenderer = new THREE.WebGLRenderer({
                canvas: legendPenguinCanvas,
                alpha: true,
                antialias: true,
            });
            legendPenguinRenderer.setSize(30, 30);
            legendPenguinRenderer.setPixelRatio(window.devicePixelRatio);

            legendPenguinScene = new THREE.Scene();

            legendPenguinCamera = new THREE.PerspectiveCamera(45, 1, 1, 1000);
            legendPenguinCamera.position.set(0, 5, 25);

            const lLight = new THREE.DirectionalLight(0xffffff, 3);
            lLight.position.set(0, 50, 50);
            legendPenguinScene.add(lLight);

            const lAmbient = new THREE.AmbientLight(0xffffff, 1);
            legendPenguinScene.add(lAmbient);

            gltfLoader.load("3d/penguin.glb", (gltf) => {
                legendPenguinModel = normalizeModel(gltf.scene, 18);
                legendPenguinModel.position.set(0, 0, 0);
                legendPenguinModel.rotation.y = -Math.PI / 8;
                legendPenguinModel.userData = {
                    baseY: 0,
                    hopHeight: 8,
                    hopSpeed: 5,
                    baseScaleY: legendPenguinModel.scale.y,
                };
                legendPenguinScene.add(legendPenguinModel);
            });
        }

        createDerivativeGraph(scene);
    }

    function normalizeModel(sceneGroup: THREE.Group, targetSize: number) {

        const box = new THREE.Box3().setFromObject(sceneGroup);
        const size = box.getSize(new THREE.Vector3());

        const maxDim = Math.max(size.x, size.y, size.z);
        if (maxDim === 0) return sceneGroup; // Avoid division by zero

        const scaleFactor = targetSize / maxDim;
        sceneGroup.scale.setScalar(scaleFactor);

        const center = box.getCenter(new THREE.Vector3());
        sceneGroup.position.x -= center.x * scaleFactor;
        sceneGroup.position.y -= center.y * scaleFactor;
        sceneGroup.position.z -= center.z * scaleFactor;

        return sceneGroup;
    }

    let genreColors: any = $derived.by(() => {
        return Object.fromEntries(
            genreList.map((genre, index) => {
                const hue = (index / genreList.length) * 360; // Distribute hues evenly
                return [genre, `hsl(${hue}, 70%, 50%)`]; 
            }),
        );
    });

    let genreLineGroups: THREE.Group[] = [];

    function createTextSprite(message: string) {
        const canvas = document.createElement("canvas");
        const context = canvas.getContext("2d");
        if (!context) return new THREE.Sprite();

        const fontSize = 40;
        context.font = `bold ${fontSize}px Arial`;
        const metrics = context.measureText(message);
        const textWidth = metrics.width;

        canvas.width = textWidth + 20;
        canvas.height = fontSize + 20;

        context.font = `bold ${fontSize}px Arial`;
        context.fillStyle = "black";
        context.fillText(message, 10, fontSize);

        const texture = new THREE.CanvasTexture(canvas);
        const spriteMaterial = new THREE.SpriteMaterial({
            map: texture,
            transparent: true,
        });
        const sprite = new THREE.Sprite(spriteMaterial);

        const aspectRatio = canvas.width / canvas.height;
        sprite.scale.set(10 * aspectRatio, 10, 1);
        return sprite;
    }

    function createDerivativeGraph(scene: THREE.Scene) {
        const xSpacing = 20;
        const yMultiplier = 50;
        const graphBaseY = FLOOR + 400;

        // Create Year Labels
        const labelsGroup = new THREE.Group();
        derivativeData.forEach((row: any, i: number) => {
            const displayYear = row.year || 1945 + i;
            if (
                displayYear % 5 === 0 ||
                i === 0 ||
                i === derivativeData.length - 1
            ) {
                const x = (i - derivativeData.length / 2) * xSpacing;
                const sprite = createTextSprite(displayYear.toString());
                sprite.scale.set(sprite.scale.x * 2, sprite.scale.y * 2, 1);
                sprite.position.set(x, graphBaseY - 60, 0);
                labelsGroup.add(sprite);
            }
        });
        scene.add(labelsGroup);

        genreList.forEach((genre, genreIndex) => {
            const group = new THREE.Group();
            group.visible = false; // Hidden by default for scroll triggering
            const points: THREE.Vector3[] = [];
            const color = genreColors[genre];

            derivativeData.forEach((row: any, i: number) => {
                const x = (i - derivativeData.length / 2) * xSpacing;
                const y = graphBaseY + row[genre] * yMultiplier;
                const z = 0;

                const pos = new THREE.Vector3(x, y, z);
                points.push(pos);


                const sphereGeo = new THREE.SphereGeometry(4, 16, 16);
                const sphereMat = new THREE.MeshStandardMaterial({
                    color,
                    metalness: 0.0,
                    roughness: 1.0,
                });
                const sphere = new THREE.Mesh(sphereGeo, sphereMat);
                sphere.position.copy(pos);
                sphere.castShadow = true; // Required for drop shadows
                sphere.receiveShadow = true;
                group.add(sphere);
            });

            const curve = new THREE.CatmullRomCurve3(points);
            const lineGeo = new THREE.TubeGeometry(
                curve,
                points.length * 4,
                1.5,
                8,
                false,
            );
            const lineMat = new THREE.MeshStandardMaterial({
                color,
                metalness: 0.0,
                roughness: 1.0,
            });
            const line = new THREE.Mesh(lineGeo, lineMat);
            line.castShadow = true;
            line.receiveShadow = true;
            group.add(line);

            scene.add(group);
            genreLineGroups.push(group);
        });
    }

    function animate() {
        // this function will be called every frame
        const delta = clock.getDelta();

        if (mixer && progress >= 50) {
            mixer.update(delta);
        }

        const elapsed = clock.getElapsedTime();

        // Toggle genre lines one by one based on scroll step
        const totalGroups = Math.max(1, genreLineGroups.length - 1);
        genreLineGroups.forEach((group, index) => {
            // Lines should appear one by one over progress 0 to 50
            group.visible = progress >= ((index / totalGroups) * 45) +2;
        });

        if (progress >= 50) {
            // Penguin Hop and Squish Animation
            hoppingModels.forEach((model) => {
                const { baseY, hopHeight, hopSpeed, baseScaleY } =
                    model.userData;
                // Vertical Hop
                model.position.y =
                    baseY + Math.abs(Math.sin(elapsed * hopSpeed)) * hopHeight;
                // Vertical Squish
                const scaleEffect =
                    1 - Math.abs(Math.sin(elapsed * hopSpeed)) * 0.2;
                model.scale.y = baseScaleY * scaleEffect;
            });
        } else {
            // Return to initial position if progress < 50
            hoppingModels.forEach((model) => {
                const { baseY, baseScaleY } = model.userData;
                model.position.y = baseY;
                model.scale.y = baseScaleY;
            });
        }

        // Orbit and other animations...
        renderer.render(scene, camera);

        // Render legend penguin
        if (
            legendPenguinRenderer &&
            legendPenguinScene &&
            legendPenguinCamera
        ) {
            if (legendPenguinModel) {
                if (progress >= 50) {
                    legendPenguinModel.position.y =
                        Math.abs(
                            Math.sin(
                                elapsed * legendPenguinModel.userData.hopSpeed,
                            ),
                        ) * legendPenguinModel.userData.hopHeight;
                    legendPenguinModel.scale.y =
                        legendPenguinModel.userData.baseScaleY *
                        (1 -
                            Math.abs(
                                Math.sin(
                                    elapsed *
                                        legendPenguinModel.userData.hopSpeed,
                                ),
                            ) *
                                0.2);
                } else {
                    legendPenguinModel.position.y =
                        legendPenguinModel.userData.baseY;
                    legendPenguinModel.scale.y =
                        legendPenguinModel.userData.baseScaleY;
                }
            }
            legendPenguinRenderer.render(
                legendPenguinScene,
                legendPenguinCamera,
            );
        }
    }
</script>

<h2>The Scrolly element can also be used to control 3D Visualizations</h2>

<Scroll bind:progress>
    <!-- Story here -->

    <div class="threejsSection">
        <p>Scroll down in this div, more genres will apear on the graph.</p>
        <br />
        <br />
        <br />
        <p>
            Some genres will have negative values, indicating a decrease in the
            number of movies released in that genre.
        </p>
        <br />
        <p>Some genres like Drama will see high amounts of fluxuation</p>
        <br />
        <p>others like Animation will see very little.</p>

        <br />
        <p>genres that have a low amount of films overall sit at 0.</p>
        <br />
        <p>
            The y axis represents the change in the number of movies released in
            that genre.
        </p>
        <br />
        <br />
        <p>The x axis represents the year.</p>
         <br />
        <br />
        <p>As more movies have been released each year many genres see much higher fluxuations, often in the positive.</p>
        <br />
        <br />
        <p>Once progress > 50, the penguins will start hopping.</p>
    </div>

    <div class="threejsSection">
        scroll down in this div will not move the camera of the scene.
    </div>

    <svelte:fragment slot="viz">
        <div class="visualization-container">
            <h2 class="viz-title">
                Yearly Derivatives of Summer Movies Released by Genre
            </h2>

            <div class="legend">
                <div class="legend-item">
                    <canvas
                        bind:this={legendPenguinCanvas}
                        width="30"
                        height="30"
                        style="display: block; margin-right: 5px;"
                    ></canvas>
                    <span class="genre-name">Penguin</span>
                </div>
                {#each genreList as genre}
                    <div class="legend-item">
                        <span
                            class="color-box"
                            style="background-color: {genreColors[genre]};"
                        ></span>
                        <span class="genre-name">{genre}</span>
                    </div>
                {/each}
            </div>

            <div bind:this={threeJSContainer}></div>
        </div>
    </svelte:fragment>
</Scroll>

<style>
    div.threejsSection {
        text-align: center;
        color: #666;
        height: 200vh; /* Make the div scrollable with a 200% view height */
        border: 2px solid #aaa;
        padding: 10px;
        margin: 10px;
    }

    .visualization-container {
        position: relative;
        width: 100%;
        height: 100%;
    }

    .viz-title {
        position: absolute;
        top: 20px;
        left: 50%;
        transform: translateX(-50%);
        color: #333;
        background: rgba(255, 255, 255, 0.85);
        padding: 12px 24px;
        border-radius: 8px;
        z-index: 10;
        margin: 0;
        font-family: sans-serif;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        text-align: center;
        white-space: nowrap;
    }

    .legend {
        position: absolute;
        bottom: 20px;
        left: 50%;
        transform: translateX(-50%);
        width: 85%;
        background: rgba(255, 255, 255, 0.85);
        padding: 15px;
        border-radius: 8px;
        display: flex;
        flex-direction: row;
        flex-wrap: wrap;
        justify-content: center;
        gap: 12px;
        z-index: 10;
        max-height: 180px;
        overflow-y: auto;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    }

    .legend-item {
        display: flex;
        align-items: center;
        gap: 10px;
        font-family: sans-serif;
        font-size: 14px;
        color: #333;
    }

    .color-box {
        width: 18px;
        height: 18px;
        border-radius: 4px;
        display: inline-block;
        border: 1px solid rgba(0, 0, 0, 0.2);
    }
</style>
