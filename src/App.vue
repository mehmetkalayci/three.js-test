<template></template>

<script setup>
import { onMounted } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";

onMounted(() => {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color("white");

  const fov = 25;
  const aspect = window.innerWidth / window.innerHeight;
  const near = 0.1;
  const far = 1000;

  const camera = new THREE.PerspectiveCamera(fov, aspect, near, far);
  camera.position.set(90, 90, 0);
  scene.add(camera);

  const renderer = new THREE.WebGLRenderer();
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  document.getElementById("app").appendChild(renderer.domElement);

  // Initial setup of camera and controls
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.25;
  controls.enableZoom = true;
  controls.update();

  const onWindowResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  };

  window.addEventListener("resize", onWindowResize);

  let largeCube = null;
  let largeCubeBox = null;
  let mixer = null;  // Mixer değişkenini global al
  let animationClip = null;  //animationClip değişkenini global al

  const updateGeometryIfIntersect = (object) => {
    if (largeCubeBox && object.geometry) {
      const objectBox = new THREE.Box3().setFromObject(object);
      if (objectBox.intersectsBox(largeCubeBox)) {
        const geometry = object.geometry;
        const positionAttribute = geometry.getAttribute('position');
        const indexAttribute = geometry.getIndex();

        if (!indexAttribute) {
          console.warn("Geometry doesn't have an index buffer. Skipping...");
          return;
        }

        const positions = positionAttribute.array;
        const indices = indexAttribute.array;
        const newPositions = [];
        const newIndices = [];

        for (let i = 0; i < indices.length; i += 3) {
          const a = new THREE.Vector3().fromBufferAttribute(positionAttribute, indices[i]);
          const b = new THREE.Vector3().fromBufferAttribute(positionAttribute, indices[i + 1]);
          const c = new THREE.Vector3().fromBufferAttribute(positionAttribute, indices[i + 2]);

          object.localToWorld(a);
          object.localToWorld(b);
          object.localToWorld(c);

          if (!largeCubeBox.containsPoint(a) && !largeCubeBox.containsPoint(b) && !largeCubeBox.containsPoint(c)) {
            const startIndex = newPositions.length / 3;
            newPositions.push(
              positions[indices[i] * 3], positions[indices[i] * 3 + 1], positions[indices[i] * 3 + 2],
              positions[indices[i + 1] * 3], positions[indices[i + 1] * 3 + 1], positions[indices[i + 1] * 3 + 2],
              positions[indices[i + 2] * 3], positions[indices[i + 2] * 3 + 1], positions[indices[i + 2] * 3 + 2]
            );
            newIndices.push(startIndex, startIndex + 1, startIndex + 2);
          }
        }

        const newGeometry = new THREE.BufferGeometry();
        newGeometry.setAttribute('position', new THREE.Float32BufferAttribute(newPositions, 3));
        newGeometry.setIndex(newIndices);
        newGeometry.computeVertexNormals();

        object.geometry.dispose();
        object.geometry = newGeometry;
      }
    }
  };

  const checkAndUpdateObjects = () => {
    scene.traverse((object) => {
      if (object.isMesh && object !== largeCube) {
        updateGeometryIfIntersect(object);
      }
    });
  };


  function animate() {
    renderer.render(scene, camera);
    requestAnimationFrame(animate);
  }

  function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  const DEFAULT_FRAME_RATE = 24; // Blender default frame rate değeri

  const loader = new GLTFLoader();
  loader.load(
    // Animasyon hazırlamak için https://www.youtube.com/watch?v=sb_FxJX4Jns
    "http://localhost:5173/untitledAnim.glb",
    async (gltf) => {
      if (gltf && gltf.scene) {
        const model = gltf.scene;


        if (gltf.animations && gltf.animations.length > 0) {
          // AnimationMixer ve AnimationClip'i oluşturun
          mixer = new THREE.AnimationMixer(model);
          const animationClip = THREE.AnimationClip.findByName(gltf.animations, 'kutuAction.001');

          // Animasyon klibi ile bir clipAction oluşturun
          const action = mixer.clipAction(animationClip);

          // İstenilen zamana ayarla
          action.play();
          const frameNumber = 83;
          // gidilecek frame göre zamanı hesapla
          const time = frameNumber / DEFAULT_FRAME_RATE;
          mixer.setTime(time);




                    
          // const totalFrames = 361;
          // for (let frameNumber = 0; frameNumber < totalFrames; frameNumber++) {
          //   const time = frameNumber / DEFAULT_FRAME_RATE;
          //   action.play();

          //   mixer.setTime(time);


          //   // Wait for 1000ms
          //   await sleep(1000);

          //   // Log frame number or perform other operations
          //   console.log(frameNumber);
            
          // }





        } else {
          console.warn("No animations found in the GLTF file");
        }
        scene.add(model);

        scene.traverse((child) => {
          if (child.isMesh) {
            child.material.wireframe = true;

            if (child.name === "kutu") {
              largeCube = child;
              const boxMaterial = new THREE.MeshBasicMaterial({
                color: 0x00aa00,
                wireframe: true,
                opacity: 0.5,
                transparent: true
              });
              largeCube.material = boxMaterial;
              largeCubeBox = new THREE.Box3().setFromObject(largeCube);
            }
          }
        });

        checkAndUpdateObjects();


      } else {
        console.error("GLTF dosyası yüklendi ancak sahne tanımsız");
      }
    },
    (xhr) => {
      console.log((xhr.loaded / xhr.total) * 100 + "% yüklendi");
    },
    (error) => {
      console.error("Bir hata oluştu", error);
    }
  );

  animate();
});
</script>
