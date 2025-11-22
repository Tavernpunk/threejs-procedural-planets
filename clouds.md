# Atmosphere and Clouds Implementation

This project creates the atmosphere and clouds using a **particle-based system** with custom shaders. Here's how it works:

## Particle Generation (scripts/atmosphere.js:42-79)
- Creates a **THREE.Points** system with thousands of particles (default 4000)
- Particles are distributed in a **spherical shell** around the planet:
  - Random points are generated in a cube `[-1, 1]³`
  - Normalized to project onto a sphere surface
  - Scaled by a random radius between `radius` and `radius + thickness`
  - This avoids pole bunching that spherical coordinates would create
- Each particle gets a random size between `minParticleSize` and `maxParticleSize`

## Cloud Texture (scripts/atmosphere.js:3-4)
- Loads `cloud.png` as a texture
- Applied to each particle sprite via the shader

## Shader Effects (index.html:336-380)

### Vertex Shader
- Positions particles in 3D space
- Sets point size for each particle

### Fragment Shader
- Uses **3D Simplex noise** (same noise function used for planet terrain) to animate clouds:
  ```glsl
  float n = simplex3((time * speed) + fragPosition / scale);
  ```
- Creates dynamic, evolving cloud patterns over time
- Calculates **lighting** based on light direction (clouds on lit side are brighter)
- Controls particle **opacity** using noise + density parameter
- Rotates UV coordinates based on noise for variation
- Multiplies by the cloud texture for realistic cloud appearance

## Animation (scripts/main.js:133-136)
- Time uniform updates each frame for noise evolution
- Atmosphere slowly rotates: `atmosphere.rotation.y += 0.0002`

## Visual Enhancement
- Uses `NormalBlending` for soft compositing
- `transparent: true` and `depthWrite: false` for proper blending
- **UnrealBloomPass** (scripts/main.js:99-103) adds a subtle glow effect

The result is a dynamic, volumetric-looking cloud layer that rotates and evolves over time, with proper lighting that responds to the directional light source.
