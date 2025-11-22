# Planar Cloud System for Spiral Galaxy Dust Clouds

This guide shows how to adapt the spherical atmosphere system to generate dust clouds along a flat plane (e.g., for a spiral galaxy) using React Three Fiber and TypeScript.

## Overview

Instead of distributing particles in a spherical shell, we'll distribute them in a disk/plane with configurable shape, density, and spiral patterns.

## Installation

```bash
npm install three @react-three/fiber @react-three/drei
npm install --save-dev @types/three
```

## 1. Particle Distribution Algorithm

### Basic Disk Distribution

```typescript
interface PlanarCloudConfig {
  particleCount: number;
  innerRadius: number;
  outerRadius: number;
  thickness: number; // Vertical thickness of the disk
  minParticleSize: number;
  maxParticleSize: number;
  spiralArms?: number; // Number of spiral arms (0 = uniform disk)
  spiralTightness?: number; // How tightly wound the spiral is
  armWidth?: number; // Width of spiral arms (0-1)
}

function generateDiskParticles(config: PlanarCloudConfig): {
  positions: Float32Array;
  sizes: Float32Array;
} {
  const {
    particleCount,
    innerRadius,
    outerRadius,
    thickness,
    minParticleSize,
    maxParticleSize,
    spiralArms = 0,
    spiralTightness = 1,
    armWidth = 0.3,
  } = config;

  const positions: number[] = [];
  const sizes: number[] = [];

  for (let i = 0; i < particleCount; i++) {
    // Random radius with bias toward outer edge (more realistic for galaxies)
    const t = Math.random();
    const radius = innerRadius + (outerRadius - innerRadius) * Math.pow(t, 0.5);

    let angle = Math.random() * Math.PI * 2;

    // Add spiral arm pattern if configured
    if (spiralArms > 0) {
      // Choose a random spiral arm
      const armIndex = Math.floor(Math.random() * spiralArms);
      const armAngle = (armIndex / spiralArms) * Math.PI * 2;
      
      // Add spiral offset based on radius
      const spiralOffset = spiralTightness * Math.log(radius / innerRadius + 1);
      
      // Add random variation within arm width
      const armVariation = (Math.random() - 0.5) * armWidth * Math.PI;
      
      angle = armAngle + spiralOffset + armVariation;
    }

    // Convert polar to cartesian
    const x = radius * Math.cos(angle);
    const z = radius * Math.sin(angle);

    // Vertical position with gaussian-like distribution (concentrated at y=0)
    const y = (Math.random() - 0.5) * thickness * Math.pow(Math.random(), 2);

    positions.push(x, y, z);

    // Random particle size
    const size = minParticleSize + Math.random() * (maxParticleSize - minParticleSize);
    sizes.push(size);
  }

  return {
    positions: new Float32Array(positions),
    sizes: new Float32Array(sizes),
  };
}
```

## 2. Shader Implementation

### Vertex Shader

```glsl
// planarCloudVert.glsl
attribute float size;
varying vec3 vPosition;
varying vec3 vWorldPosition;

void main() {
  vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
  gl_PointSize = size * (300.0 / -mvPosition.z); // Scale with distance
  gl_Position = projectionMatrix * mvPosition;
  
  vPosition = position;
  vWorldPosition = (modelMatrix * vec4(position, 1.0)).xyz;
}
```

### Fragment Shader with 3D Simplex Noise

```glsl
// planarCloudFrag.glsl
uniform float time;
uniform float speed;
uniform float opacity;
uniform float density;
uniform float scale;
uniform vec3 color;
uniform sampler2D pointTexture;
uniform vec3 lightDirection;

varying vec3 vPosition;
varying vec3 vWorldPosition;

// Simplex 3D Noise by Ian McEwan, Ashima Arts
vec4 permute(vec4 x) {
  return mod(((x * 34.0) + 1.0) * x, 289.0);
}

vec4 taylorInvSqrt(vec4 r) {
  return 1.79284291400159 - 0.85373472095314 * r;
}

float simplex3(vec3 v) { 
  const vec2 C = vec2(1.0/6.0, 1.0/3.0);
  const vec4 D = vec4(0.0, 0.5, 1.0, 2.0);

  // First corner
  vec3 i  = floor(v + dot(v, C.yyy));
  vec3 x0 = v - i + dot(i, C.xxx);

  // Other corners
  vec3 g = step(x0.yzx, x0.xyz);
  vec3 l = 1.0 - g;
  vec3 i1 = min(g.xyz, l.zxy);
  vec3 i2 = max(g.xyz, l.zxy);

  vec3 x1 = x0 - i1 + 1.0 * C.xxx;
  vec3 x2 = x0 - i2 + 2.0 * C.xxx;
  vec3 x3 = x0 - 1.0 + 3.0 * C.xxx;

  // Permutations
  i = mod(i, 289.0);
  vec4 p = permute(permute(permute(
    i.z + vec4(0.0, i1.z, i2.z, 1.0))
    + i.y + vec4(0.0, i1.y, i2.y, 1.0))
    + i.x + vec4(0.0, i1.x, i2.x, 1.0));

  // Gradients
  float n_ = 1.0/7.0;
  vec3 ns = n_ * D.wyz - D.xzx;

  vec4 j = p - 49.0 * floor(p * ns.z * ns.z);

  vec4 x_ = floor(j * ns.z);
  vec4 y_ = floor(j - 7.0 * x_);

  vec4 x = x_ * ns.x + ns.yyyy;
  vec4 y = y_ * ns.x + ns.yyyy;
  vec4 h = 1.0 - abs(x) - abs(y);

  vec4 b0 = vec4(x.xy, y.xy);
  vec4 b1 = vec4(x.zw, y.zw);

  vec4 s0 = floor(b0) * 2.0 + 1.0;
  vec4 s1 = floor(b1) * 2.0 + 1.0;
  vec4 sh = -step(h, vec4(0.0));

  vec4 a0 = b0.xzyw + s0.xzyw * sh.xxyy;
  vec4 a1 = b1.xzyw + s1.xzyw * sh.zzww;

  vec3 p0 = vec3(a0.xy, h.x);
  vec3 p1 = vec3(a0.zw, h.y);
  vec3 p2 = vec3(a1.xy, h.z);
  vec3 p3 = vec3(a1.zw, h.w);

  // Normalize gradients
  vec4 norm = taylorInvSqrt(vec4(dot(p0,p0), dot(p1,p1), dot(p2,p2), dot(p3,p3)));
  p0 *= norm.x;
  p1 *= norm.y;
  p2 *= norm.z;
  p3 *= norm.w;

  // Mix final noise value
  vec4 m = max(0.6 - vec4(dot(x0,x0), dot(x1,x1), dot(x2,x2), dot(x3,x3)), 0.0);
  m = m * m;
  return 42.0 * dot(m * m, vec4(dot(p0,x0), dot(p1,x1), dot(p2,x2), dot(p3,x3)));
}

vec2 rotateUV(vec2 uv, float rotation) {
  float mid = 0.5;
  return vec2(
    cos(rotation) * (uv.x - mid) + sin(rotation) * (uv.y - mid) + mid,
    cos(rotation) * (uv.y - mid) - sin(rotation) * (uv.x - mid) + mid
  );
}

void main() {
  // Calculate lighting (optional - for top-down lighting on galaxy plane)
  vec3 normal = vec3(0.0, 1.0, 0.0); // Plane normal pointing up
  vec3 L = normalize(lightDirection);
  float light = max(0.3, dot(normal, L)); // Min ambient of 0.3

  // Animated noise for cloud variation
  float n = simplex3((time * speed) + vPosition / scale);
  float alpha = opacity * clamp(n + density, 0.0, 1.0);

  // Rotate texture based on noise for variation
  vec2 rotCoords = rotateUV(gl_PointCoord, n);
  
  vec4 texColor = texture2D(pointTexture, rotCoords);
  gl_FragColor = vec4(light * color, alpha) * texColor;
}
```

## 3. React Three Fiber Component

```typescript
// PlanarClouds.tsx
import { useRef, useMemo, useEffect } from 'react';
import { useFrame, useLoader } from '@react-three/fiber';
import * as THREE from 'three';

interface PlanarCloudsProps {
  config: PlanarCloudConfig;
  textureUrl: string;
  color?: THREE.Color;
  speed?: number;
  opacity?: number;
  density?: number;
  scale?: number;
  lightDirection?: THREE.Vector3;
  rotationSpeed?: number;
}

const vertexShader = `
  attribute float size;
  varying vec3 vPosition;
  varying vec3 vWorldPosition;

  void main() {
    vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
    gl_PointSize = size * (300.0 / -mvPosition.z);
    gl_Position = projectionMatrix * mvPosition;
    
    vPosition = position;
    vWorldPosition = (modelMatrix * vec4(position, 1.0)).xyz;
  }
`;

const fragmentShader = `
  uniform float time;
  uniform float speed;
  uniform float opacity;
  uniform float density;
  uniform float scale;
  uniform vec3 color;
  uniform sampler2D pointTexture;
  uniform vec3 lightDirection;

  varying vec3 vPosition;
  varying vec3 vWorldPosition;

  // [Include full simplex3 function here - same as above]
  
  vec2 rotateUV(vec2 uv, float rotation) {
    float mid = 0.5;
    return vec2(
      cos(rotation) * (uv.x - mid) + sin(rotation) * (uv.y - mid) + mid,
      cos(rotation) * (uv.y - mid) - sin(rotation) * (uv.x - mid) + mid
    );
  }

  void main() {
    vec3 normal = vec3(0.0, 1.0, 0.0);
    vec3 L = normalize(lightDirection);
    float light = max(0.3, dot(normal, L));

    float n = simplex3((time * speed) + vPosition / scale);
    float alpha = opacity * clamp(n + density, 0.0, 1.0);

    vec2 rotCoords = rotateUV(gl_PointCoord, n);
    vec4 texColor = texture2D(pointTexture, rotCoords);
    gl_FragColor = vec4(light * color, alpha) * texColor;
  }
`;

export function PlanarClouds({
  config,
  textureUrl,
  color = new THREE.Color(0xffffff),
  speed = 0.03,
  opacity = 0.35,
  density = 0,
  scale = 8,
  lightDirection = new THREE.Vector3(0, 1, 0),
  rotationSpeed = 0.0002,
}: PlanarCloudsProps) {
  const pointsRef = useRef<THREE.Points>(null);
  const texture = useLoader(THREE.TextureLoader, textureUrl);

  // Generate particle geometry
  const geometry = useMemo(() => {
    const { positions, sizes } = generateDiskParticles(config);
    
    const geo = new THREE.BufferGeometry();
    geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geo.setAttribute('size', new THREE.BufferAttribute(sizes, 1));
    
    return geo;
  }, [config]);

  // Create shader material
  const material = useMemo(() => {
    return new THREE.ShaderMaterial({
      uniforms: {
        time: { value: 0 },
        speed: { value: speed },
        opacity: { value: opacity },
        density: { value: density },
        scale: { value: scale },
        color: { value: color },
        pointTexture: { value: texture },
        lightDirection: { value: lightDirection },
      },
      vertexShader,
      fragmentShader,
      transparent: true,
      depthWrite: false,
      blending: THREE.NormalBlending,
    });
  }, [speed, opacity, density, scale, color, texture, lightDirection]);

  // Animate
  useFrame((state) => {
    if (pointsRef.current) {
      material.uniforms.time.value = state.clock.getElapsedTime();
      pointsRef.current.rotation.y += rotationSpeed;
    }
  });

  // Cleanup
  useEffect(() => {
    return () => {
      geometry.dispose();
      material.dispose();
    };
  }, [geometry, material]);

  return <points ref={pointsRef} geometry={geometry} material={material} />;
}
```

## 4. Usage Example

```typescript
// App.tsx
import { Canvas } from '@react-three/fiber';
import { OrbitControls } from '@react-three/drei';
import { PlanarClouds } from './PlanarClouds';
import * as THREE from 'three';

function App() {
  const galaxyConfig: PlanarCloudConfig = {
    particleCount: 10000,
    innerRadius: 5,
    outerRadius: 50,
    thickness: 2,
    minParticleSize: 20,
    maxParticleSize: 80,
    spiralArms: 4,
    spiralTightness: 0.5,
    armWidth: 0.4,
  };

  return (
    <div style={{ width: '100vw', height: '100vh' }}>
      <Canvas camera={{ position: [0, 80, 0], fov: 75 }}>
        {/* Lighting */}
        <ambientLight intensity={0.5} />
        
        {/* Galaxy dust clouds */}
        <PlanarClouds
          config={galaxyConfig}
          textureUrl="/cloud.png"
          color={new THREE.Color(0x88ccff)}
          speed={0.02}
          opacity={0.4}
          density={0.1}
          scale={10}
          lightDirection={new THREE.Vector3(0, -1, 0)}
          rotationSpeed={0.0001}
        />
        
        {/* Camera controls */}
        <OrbitControls enablePan={false} />
      </Canvas>
    </div>
  );
}

export default App;
```

## 5. Configuration Options

### Basic Disk
```typescript
const basicDisk: PlanarCloudConfig = {
  particleCount: 5000,
  innerRadius: 10,
  outerRadius: 40,
  thickness: 1,
  minParticleSize: 30,
  maxParticleSize: 60,
  spiralArms: 0, // No spiral pattern
};
```

### Spiral Galaxy
```typescript
const spiralGalaxy: PlanarCloudConfig = {
  particleCount: 15000,
  innerRadius: 5,
  outerRadius: 60,
  thickness: 3,
  minParticleSize: 20,
  maxParticleSize: 100,
  spiralArms: 3,
  spiralTightness: 0.8,
  armWidth: 0.3,
};
```

### Dense Nebula
```typescript
const nebula: PlanarCloudConfig = {
  particleCount: 20000,
  innerRadius: 0,
  outerRadius: 30,
  thickness: 5,
  minParticleSize: 40,
  maxParticleSize: 120,
  spiralArms: 0,
};
```

## 6. Advanced Features

### Multiple Cloud Layers
```typescript
<group>
  {/* Dense core */}
  <PlanarClouds
    config={{ ...config, outerRadius: 20, particleCount: 5000 }}
    opacity={0.6}
    color={new THREE.Color(0xffaa44)}
  />
  
  {/* Sparse outer layer */}
  <PlanarClouds
    config={{ ...config, innerRadius: 18, outerRadius: 50, particleCount: 3000 }}
    opacity={0.3}
    color={new THREE.Color(0x4488ff)}
  />
</group>
```

### Density Falloff with Radius
Modify the particle generation to reduce density at outer edges:

```typescript
// In generateDiskParticles, after calculating radius:
const densityFalloff = 1 - Math.pow(t, 2); // More particles toward center
if (Math.random() > densityFalloff) {
  continue; // Skip this particle
}
```

### Rotation Speed Based on Radius
Add differential rotation like real galaxies:

```typescript
// In useFrame hook
useFrame((state) => {
  if (pointsRef.current) {
    const positions = geometry.attributes.position.array;
    
    for (let i = 0; i < positions.length; i += 3) {
      const x = positions[i];
      const z = positions[i + 2];
      const radius = Math.sqrt(x * x + z * z);
      
      // Rotate faster near center (Keplerian rotation)
      const rotSpeed = rotationSpeed * (outerRadius / (radius + 1));
      const angle = Math.atan2(z, x) + rotSpeed;
      
      positions[i] = radius * Math.cos(angle);
      positions[i + 2] = radius * Math.sin(angle);
    }
    
    geometry.attributes.position.needsUpdate = true;
    material.uniforms.time.value = state.clock.getElapsedTime();
  }
});
```

## 7. Performance Tips

1. **Use instancing for very large particle counts (>50k)**
2. **Implement LOD** - reduce particle count/size when camera is far away
3. **Frustum culling** - don't render particles outside camera view
4. **Texture atlas** - use multiple cloud shapes in one texture
5. **GPU culling** - discard particles with alpha < threshold in shader

## 8. Cloud Texture

Create or use a cloud texture. A simple approach:

```typescript
// Generate procedural cloud texture
function createCloudTexture(size = 128): THREE.Texture {
  const canvas = document.createElement('canvas');
  canvas.width = size;
  canvas.height = size;
  const ctx = canvas.getContext('2d')!;

  const gradient = ctx.createRadialGradient(
    size / 2, size / 2, 0,
    size / 2, size / 2, size / 2
  );
  gradient.addColorStop(0, 'rgba(255, 255, 255, 1)');
  gradient.addColorStop(0.5, 'rgba(255, 255, 255, 0.5)');
  gradient.addColorStop(1, 'rgba(255, 255, 255, 0)');

  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, size, size);

  return new THREE.CanvasTexture(canvas);
}
```

## Summary

This implementation provides:
- ✅ Configurable disk/planar particle distribution
- ✅ Optional spiral arm patterns for galaxies
- ✅ Animated noise for dynamic cloud movement
- ✅ Customizable appearance (color, opacity, density)
- ✅ TypeScript type safety
- ✅ React Three Fiber integration
- ✅ Performant shader-based rendering

The system is highly configurable and can be adapted for various use cases from dense nebulae to sparse dust clouds in spiral galaxies.
