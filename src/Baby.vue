<template>
    <canvas ref="renderCanvas"></canvas>
  </template>
  
  <script setup>
  import { ref, onMounted } from 'vue';
  import * as BABYLON from '@babylonjs/core';
  import '@babylonjs/loaders/glTF';
  
  const renderCanvas = ref(null);
  
  onMounted(() => {
    const canvas = renderCanvas.value;
    const engine = new BABYLON.Engine(canvas, true);
    const scene = new BABYLON.Scene(engine);
  
    const camera = new BABYLON.ArcRotateCamera("camera", -Math.PI / 2, Math.PI / 2.5, 10, BABYLON.Vector3.Zero(), scene);
    camera.attachControl(canvas, true);
  
    const light = new BABYLON.HemisphericLight("light", new BABYLON.Vector3(0, 1, 0), scene);
  
    let largeCube = null;
  
    BABYLON.SceneLoader.ImportMesh("", "http://localhost:5173/public/", "untitled.glb", scene, function (meshes) {
      console.log("Loaded meshes:", meshes);
  
      meshes.forEach((mesh) => {
        console.log("Mesh name:", mesh.name);
        
        if (mesh.name === "kutu") {
          largeCube = mesh;
          largeCube.material = new BABYLON.StandardMaterial("largeCubeMaterial", scene);
          largeCube.material.wireframe = true;
          largeCube.material.alpha = 0.5;
          console.log("Large cube found:", largeCube);
        } else if (mesh.geometry) {
          updateGeometryIfIntersect(mesh, largeCube);
        }
      });
    });
  
    function updateGeometryIfIntersect(mesh, largeCube) {
      if (!largeCube) return;
  
      const positions = mesh.getVerticesData(BABYLON.VertexBuffer.PositionKind);
      const indices = mesh.getIndices();
      const newPositions = [];
      const newIndices = [];
  
      const worldMatrix = mesh.computeWorldMatrix(true);
      const largeCubeBoundingBox = largeCube.getBoundingInfo().boundingBox;
  
      for (let i = 0; i < indices.length; i += 3) {
        const a = new BABYLON.Vector3(positions[indices[i]*3], positions[indices[i]*3+1], positions[indices[i]*3+2]);
        const b = new BABYLON.Vector3(positions[indices[i+1]*3], positions[indices[i+1]*3+1], positions[indices[i+1]*3+2]);
        const c = new BABYLON.Vector3(positions[indices[i+2]*3], positions[indices[i+2]*3+1], positions[indices[i+2]*3+2]);
  
        BABYLON.Vector3.TransformCoordinatesToRef(a, worldMatrix, a);
        BABYLON.Vector3.TransformCoordinatesToRef(b, worldMatrix, b);
        BABYLON.Vector3.TransformCoordinatesToRef(c, worldMatrix, c);
  
        if (!isPointInside(a, largeCubeBoundingBox) && 
            !isPointInside(b, largeCubeBoundingBox) && 
            !isPointInside(c, largeCubeBoundingBox)) {
          const startIndex = newPositions.length / 3;
          newPositions.push(
            ...a.asArray(), ...b.asArray(), ...c.asArray()
          );
          newIndices.push(startIndex, startIndex + 1, startIndex + 2);
        }
      }
  
      if (newPositions.length > 0) {
        const newMesh = new BABYLON.Mesh(mesh.name + "_cut", scene);
        const vertexData = new BABYLON.VertexData();
        vertexData.positions = newPositions;
        vertexData.indices = newIndices;
        vertexData.applyToMesh(newMesh);
  
        newMesh.material = mesh.material;
        mesh.dispose();
      } else {
        console.log("Mesh completely inside large cube:", mesh.name);
        mesh.dispose();
      }
    }
  
    function isPointInside(point, boundingBox) {
      return boundingBox.intersectsPoint(point);
    }
  
    scene.debugLayer.show();
  
    engine.runRenderLoop(() => {
      scene.render();
    });
  
    window.addEventListener("resize", () => {
      engine.resize();
    });
  });
  </script>
  
  <style scoped>
  canvas {
    width: 100%;
    height: 100vh;
  }
  </style>