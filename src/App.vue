    <template></template>

    <script setup>
    import { onMounted } from "vue";
    import * as THREE from "three";
    import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
    import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";

    onMounted(() => {
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(
        75,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
    );
    const renderer = new THREE.WebGLRenderer();

    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);

    document.getElementById("app").appendChild(renderer.domElement);

    // Initial setup of camera and controls
    camera.position.z = 5;

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.25;
    controls.enableZoom = true;

    const animate = () => {
        requestAnimationFrame(animate);

        controls.update();
        renderer.render(scene, camera);
    };

    animate();

    const onWindowResize = () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    };

    window.addEventListener("resize", onWindowResize);

    let largeCube = null;
    let largeCubeBox = null;

    // Function to update vertices of an object if it intersects with the large box
    const updateVerticesIfIntersect = (object) => {
        if (largeCubeBox) {
        const objectBox = new THREE.Box3().setFromObject(object);
        if (objectBox.intersectsBox(largeCubeBox)) {
            const geometry = object.geometry;
            const vertices = geometry.attributes.position.array;
            const newVertices = [];
            for (let i = 0; i < vertices.length; i += 3) {
            const vertex = new THREE.Vector3(
                vertices[i],
                vertices[i + 1],
                vertices[i + 2]
            );
            object.localToWorld(vertex); // Convert to world coordinates
            if (!largeCubeBox.containsPoint(vertex)) {
                newVertices.push(vertices[i], vertices[i + 1], vertices[i + 2]);
            } else {
                console.log(vertex);
            }
            }
            geometry.setAttribute(
            "position",
            new THREE.Float32BufferAttribute(newVertices, 3)
            );
            geometry.attributes.position.needsUpdate = true;
        }
        }
    };

    // Function to check and update all objects in the scene
    const checkAndUpdateObjects = () => {
        scene.children.forEach((object) => {
        if (object !== largeCube && object.geometry) {
            updateVerticesIfIntersect(object);
        }
        });
    };

    // Load GLB file and extract the "kutu" object
    const loader = new GLTFLoader();
    loader.load(
        "http://localhost:5173/public/test.glb", // Path to your GLB file
        (gltf) => {
        if (gltf && gltf.scene) {
            const objectsToCheck = [];

            gltf.scene.children.forEach((child) => {
            if (child.isMesh) {
                if (child.userData.name === "kutu") {
                largeCube = child;
                child.material.wireframe = true; // Set the material to wireframe mode
                largeCubeBox = new THREE.Box3().setFromObject(largeCube);
                scene.add(largeCube);
                } else {
                objectsToCheck.push(child);
                }
            }
            });

            // Add objects to the scene and check for intersections
            objectsToCheck.forEach((object) => scene.add(object));
            checkAndUpdateObjects();
        } else {
            console.error("GLTF file loaded but scene is undefined");
        }
        },
        (xhr) => {
        console.log((xhr.loaded / xhr.total) * 100 + "% loaded");
        },
        (error) => {
        console.error("An error happened", error);
        }
    );
    });
    </script>

    <style>
    #app {
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    margin: 0;
    padding: 0;
    }
    </style>
