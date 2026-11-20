# Professional Framework Migration Guide: React 18 + Vite + React Three Fiber (R3F)

> **Architectural Blueprint for Yadhukrishnan MA's Robotics Intelligence Lab**  
> Prepared as part of the **Pack C** upgrade roadmap.

---

## 1. Executive Rationale

The current single-file implementation (`index.html`) achieves zero-downtime, maximum execution speed, and instant GitHub Pages deployment. However, as the robotics platform scales to include live WebRTC teleoperation, real-time Foxglove/rosbridge WebSockets, and deep neural net visualizers, transitioning to **React + React Three Fiber (R3F) + Vite** provides:

1. **Declarative 3D Scene Graph**: Eliminates imperative boilerplate (`scene.add()`, manual disposal loops, matrix updates) by representing 3D entities as pure React components (`<mesh>`, `<primitive>`, `<group>`).
2. **Automatic WebGL Memory Lifecycle**: R3F automatically calls `.dispose()` on geometries, materials, and textures when components unmount, eliminating GPU memory leaks.
3. **Reactive Telemetry State (Zustand)**: Decouples high-frequency robot joint states and LiDAR sensor streams from DOM re-renders using transient state subscriptions.
4. **Drei Ecosystem Superpowers**: Built-in access to `useGLTF`, `useAnimations`, `Html` 3D-anchored DOM overlays, `OrbitControls`, `Float`, and `MeshTransmissionMaterial`.

---

## 2. Recommended Directory Architecture

```
yadhu-robotics-lab/
├── public/
│   ├── assets/
│   │   ├── models/
│   │   │   ├── Xbot.glb
│   │   │   └── urdf_arm/
│   │   ├── textures/
│   │   └── audio/
├── src/
│   ├── components/
│   │   ├── 3d/
│   │   │   ├── CanvasStage.tsx          (Global Canvas with EffectComposer & ToneMapping)
│   │   │   ├── HolographicRobot.tsx     (R3F Humanoid with Custom GLSL Shader)
│   │   │   ├── EvolutionTrack3D.tsx     (16-Era Curved Spline Cinema Plates)
│   │   │   ├── UrdfManipulator3D.tsx    (Articulated 6-Axis Robot Arm with FK)
│   │   │   └── LidarPointCloud3D.tsx    (InstancedBufferGeometry Point Cloud)
│   │   ├── ui/
│   │   │   ├── Nav2Simulator.tsx        (2D Canvas MinHeap A* Diff-Drive Engine)
│   │   │   ├── RosArchitectureGraph.tsx (React Flow Interactive Node/Topic Graph)
│   │   │   ├── TeleoperationCockpit.tsx (Multi-Camera Viewports & Joint Bars)
│   │   │   ├── ProjectCardGrid.tsx      (12 Projects with Framer Motion layoutId)
│   │   │   └── ProjectDossierModal.tsx  (Full-Screen Morphing Case Study)
│   │   └── audio/
│   │       └── SoundManager.ts          (Web Audio Spatial Soundscape Engine)
│   ├── hooks/
│   │   ├── useKinematics.ts             (Forward & Inverse Kinematics calculations)
│   │   ├── useEraScroll.ts              (Lenis + GSAP ScrollTrigger bridge)
│   │   └── useGpuTier.ts                (detect-gpu adaptive performance tiers)
│   ├── store/
│   │   └── useLabStore.ts               (Zustand store for robot telemetry & HUD state)
│   ├── styles/
│   │   └── globals.css                  (Tailwind CSS v4 Design Tokens)
│   ├── App.tsx
│   └── main.tsx
├── index.html
├── package.json
├── tsconfig.json
└── vite.config.ts
```

---

## 3. Package Dependencies & Setup

```bash
# 1. Initialize Vite + React + TypeScript
npm create vite@latest yadhu-robotics-lab -- --template react-ts
cd yadhu-robotics-lab

# 2. Install 3D & Graphics Core
npm install three @types/three @react-three/fiber @react-three/drei @react-three/postprocessing

# 3. Install Animation & UX Engines
npm install gsap lenis framer-motion @xyflow/react zustand

# 4. Install Robotics & Utility Libraries
npm install urdf-loader detect-gpu lucide-react clsx tailwind-merge
```

---

## 4. Key Component Blueprints

### A. Holographic Robot Component (`HolographicRobot.tsx`)

```tsx
import React, { useRef, useMemo } from 'react';
import { useFrame } from '@react-three/fiber';
import { useGLTF, useAnimations } from '@react-three/drei';
import * as THREE from 'three';

export function HolographicRobot({ isHologram = true, activeAnim = 'idle' }: { isHologram?: boolean, activeAnim?: string }) {
  const group = useRef<THREE.Group>(null);
  const { scene, animations } = useGLTF('/assets/models/Xbot.glb');
  const { actions } = useAnimations(animations, group);

  // Holographic GLSL Shader
  const hologramShader = useMemo(() => new THREE.ShaderMaterial({
    uniforms: {
      uTime: { value: 0 },
      uColor: { value: new THREE.Color('#00e5ff') }
    },
    vertexShader: `
      varying vec3 vNormal;
      varying vec3 vWorldPosition;
      void main() {
        vNormal = normalize(normalMatrix * normal);
        vec4 wp = modelMatrix * vec4(position, 1.0);
        vWorldPosition = wp.xyz;
        gl_Position = projectionMatrix * viewMatrix * wp;
      }
    `,
    fragmentShader: `
      uniform float uTime;
      uniform vec3 uColor;
      varying vec3 vNormal;
      varying vec3 vWorldPosition;
      void main() {
        vec3 normal = normalize(vNormal);
        vec3 viewDir = normalize(cameraPosition - vWorldPosition);
        float fresnel = pow(clamp(1.0 - dot(viewDir, normal), 0.0, 1.0), 2.5);
        float scanline = sin(vWorldPosition.y * 55.0 - uTime * 7.0) * 0.12 + 0.88;
        gl_FragColor = vec4(uColor * (fresnel * 0.8 + 0.2) * scanline, fresnel * 0.85 + 0.15);
      }
    `,
    transparent: true,
    depthWrite: false,
    blending: THREE.AdditiveBlending
  }), []);

  // Sync animations
  React.useEffect(() => {
    actions[activeAnim]?.reset().fadeIn(0.35).play();
    return () => { actions[activeAnim]?.fadeOut(0.35); };
  }, [activeAnim, actions]);

  useFrame((state, delta) => {
    hologramShader.uniforms.uTime.value = state.clock.getElapsedTime();
  });

  return (
    <primitive
      ref={group}
      object={scene}
      position={[0, 0, 0]}
    />
  );
}
```

### B. Reactive Telemetry Store (`useLabStore.ts`)

```typescript
import { create } from 'zustand';

interface LabState {
  linearVelocity: number;
  angularVelocity: number;
  wheelLeft: number;
  wheelRight: number;
  heading: number;
  isAudioEnabled: boolean;
  activeProjectDossier: string | null;
  setTelemetry: (v: number, w: number, vl: number, vr: number, theta: number) => void;
  toggleAudio: () => void;
  openDossier: (projectId: string) => void;
  closeDossier: () => void;
}

export const useLabStore = create<LabState>((set) => ({
  linearVelocity: 0,
  angularVelocity: 0,
  wheelLeft: 0,
  wheelRight: 0,
  heading: 0,
  isAudioEnabled: false,
  activeProjectDossier: null,
  setTelemetry: (v, w, vl, vr, theta) => set({ linearVelocity: v, angularVelocity: w, wheelLeft: vl, wheelRight: vr, heading: theta }),
  toggleAudio: () => set((state) => ({ isAudioEnabled: !state.isAudioEnabled })),
  openDossier: (id) => set({ activeProjectDossier: id }),
  closeDossier: () => set({ activeProjectDossier: null })
}));
```

---

## 5. Summary & Deployment

* **Current Architecture**: The standalone single-file `index.html` remains the active production runtime.
* **Transition Trigger**: When ready to deploy to Vercel/Netlify with team contributions or automated CI/CD pipelines, execute the scaffold above. All Three.js shaders, math algorithms ($A^*$, unicycle kinematics, URDF matrices), and audio synthetics built in `index.html` translate 1:1 into these modular TypeScript components.
