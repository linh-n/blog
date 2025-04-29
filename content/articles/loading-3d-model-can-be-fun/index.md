---
title: "Progressive loading of 3D models"
date: "2025-03-08"
lastmod: "2025-04-29"
summary: "An approach to enhance the user experience during 3D model downloads by progressively rendering the model as data is received."
description: "Progressive loading of 3D models to improve user experience."
tags: ["3d", "threejs", "typescript", "shader"]
showTags: true
toc: false
math: false
readTime: false
hideBackToTop: false
---

![Loading progress](images/preview.gif#full "Demo of loading progress")

## Overview

This article is not about detailed implementation, but rather a high-level overview of how to make loading 3D models more fun. The idea is to render the model as it is being downloaded, rather than waiting for the entire model to be downloaded before rendering it. You are expected to have moderate to advanced knowledge of 3D graphics, data structure and server-client communication.

Sample code is written in TypeScript and rendered with Three.js, but the concepts can be applied to any language or framework.

### Why?

The main reason for this is to make the user experience more enjoyable. Waiting for a 3D model to download can be boring, and by rendering the model as it is being downloaded, we can make the waiting time more engaging.

Other solutions do exist, such as [Level of Detail](https://threejs.org/docs/#api/en/objects/LOD), [pop buffer](https://github.com/thibauts/pop-buffer) or [Nexus](https://vcg.isti.cnr.it/vcgtools/nexus/), but all of them actually increases the time to load the full model.

### How?

The basic idea is to use a technique called progressive loading. This involves breaking the model into smaller meaningful chunks and streaming each chunk separately. As each chunk is downloaded, we can render the current data we have received.

Geometry data consists of vertices and triangle indices. Vertices are points in 3D space, and triangle indices are used to define the triangles that make up the model. Each triangle is defined by three vertices.

My idea is to use the vertices to create a point cloud, so more vertices the client receives, more points will be rendered demonstrating the progress and the overall shape of the model. After everything is downloaded, we can replace the points with the actual model.

As this approach requires careful preparation of the data format, data transfer, and rendering, coordination between the server and client is essential. The server will need to send the data in a specific format, and the client will need to be able to parse and render the data as it is being received.

## Data format

Geometry data is just a bunch of numbers, so depending on your use, you can use indexed or non-indexed geometry. Indexed geometry is more efficient in terms of memory usage, but non-indexed geometry is easier to work with. For this example, we will use non-indexed geometry. Data will be serialized to an array of numbers with every three consecutive numbers representing a vertex.

So for example, if we have 3 vertices with coordinates of `{ x: 1.0, y: 2.0, z: 3.0}, { x: 2.0, y: 1.0, z: 0.0}, { x: 0.0, y: 2.0, z: 1.0 }`, the data will look like this, where each number is a float and occupies 4 bytes:

```goat
+----------- v1 ------------+ +----------- v2 ------------+ +----------- v3 ------------+
|                           | |                           | |                           |
+-------+ +-------+ +-------+ +-------+ +-------+ +-------+ +-------+ +-------+ +-------+
|  1.0  | |  2.0  | |  3.0  | |  2.0  | |  1.0  | |  1.0  | |  0.0  | |  2.0  | |  1.0  |
+-------+ +-------+ +-------+ +-------+ +-------+ +-------+ +-------+ +-------+ +-------+
```

## Data transfer

3D models are usually stored in binary files (.stl, .obj, .fbx, .gltf, etc.) which are not very suitable for streaming. So you can either parse the whole file on the server at request time and send the data in the format described above, or you can process the file beforehand and store the pre-processed and stream the data as-is when requested. For larger files, the second option is preferred as it will save processing time and resource on the server. Piping the data stream to a compression stream is recommended to reduce the size of the data being sent over the network, although it can be harsh on server CPU. 

After preparing the array of Float32, we need to convert it into a `ReadableStream<Uint8Array>` and send it with `Transfer-Encoding: chunked` header. There will be high chance that the data will be broken in the middle. Let's take the previous example and assume that the data is being sent in 4 chunks.

```goat
+----------- v1 ------------+ +----------- v2 ------------+ +----------- v3 ------------+
|                           | |                           | |                           |
+-------+ +-------+ +-------+ +-------+ +-------+ +-------+ +-------+ +-------+ +-------+
|  1.0  | |  2.0  | |  3.0  | |  2.0  | |  1.0  | |  1.0  | |  0.0  | |  2.0  | |  1.0  |
+-------+ +-------+ +-------+ +-------+ +-------+ +-------+ +-------+ +-------+ +-------+

 * * * *   * * * *   * *

         chunk 1
                         * *   * * * *   * * * *   *

                                    chunk 2
                                                     * * *   * * * *   * * *

                                                             chunk 3
                                                                             *   * * * *

                                                                                chunk 4
```

Each dot represents a byte, and since we are using Float32 for numbers, each number will need 4 bytes.

The first chunk will not contain enough data to render a point, so the client will put the bytes into a buffer and wait for the next chunk. It will be concatenated to the next chunk 2, and now we take the first 12 bytes and render the first point. The rest of chunk 2 will replace the buffer and so on.

Here's a piece of code that demonstrates how to process incoming data on the client side:

```typescript
export const fetchBinaryStream = async <T extends Float32Array | Uint32Array>({
  url,
  arrayType,
  bytesPerElement,
  onChunkProcessed,
  onComplete,
  signal,
}: {
  url: string;
  arrayType: new (
    buffer: ArrayBufferLike,
    byteOffset: number,
    length: number
  ) => T;
  bytesPerElement: number;
  onChunkProcessed: (array: T) => void;
  onComplete?: () => void;
  signal?: AbortSignal;
}) => {
  const response = await fetch(url, { signal });
  const reader = response.body?.getReader();
  let partialBuffer: Uint8Array | null = null;

  if (reader) {
    while (true) {
      const { done, value } = await reader.read();

      if (done) {
        // Process any remaining data in partialBuffer if stream is done
        if (partialBuffer && partialBuffer.length >= bytesPerElement) {
          const typedArray = new arrayType(
            partialBuffer.buffer,
            partialBuffer.byteOffset,
            Math.floor(partialBuffer.length / bytesPerElement)
          );
          onChunkProcessed(typedArray);
        }
        break;
      }

      if (value) {
        let processBuffer: Uint8Array;

        if (partialBuffer) {
          // Combine previous partial buffer with the new chunk
          processBuffer = new Uint8Array(partialBuffer.length + value.length);
          processBuffer.set(partialBuffer, 0);
          processBuffer.set(value, partialBuffer.length);
          partialBuffer = null;
        } else {
          processBuffer = value;
        }

        // Calculate how many complete typed values we have
        const completeElements = Math.floor(
          processBuffer.length / bytesPerElement
        );
        const completeBytes = completeElements * bytesPerElement;

        // Process complete typed values
        if (completeElements > 0) {
          const typedArray = new arrayType(
            processBuffer.buffer,
            processBuffer.byteOffset,
            completeElements
          );
          onChunkProcessed(typedArray);
        }

        // Store any remaining partial bytes for the next chunk
        if (completeBytes < processBuffer.length) {
          partialBuffer = processBuffer.slice(completeBytes);
        }
      }
    }
  }

  if (onComplete) {
    onComplete();
  }
};
```

## Rendering

### Simple approach

The simplest approach is to use `THREE.Points` to render the points. As we use chunked data, relevant information about the number of vertices must be communicated to the client beforehand via a separate api request.

After having the number of vertices, we can create a `THREE.BufferGeometry` and set the position attribute to the `Float32Array` we received. The `THREE.Points` will take care of rendering the points for us.

```ts
const pointsGeometry = new THREE.BufferGeometry();
const pointsPositions = new Float32Array(verticesCount * 3);
pointsGeometry.setAttribute(
  "position",
  new THREE.BufferAttribute(pointsPositions, 3)
);
```

After this initialization, we can use the `fetchBinaryStream` function to receive the data and update the geometry:

```ts
let index = 0; // Current index in the vertices array

await fetchBinaryStream({
  // You must implement your own API to serve the data
  url: `/api/stream/vertices/${verticesId}`,
  arrayType: Float32Array,
  bytesPerElement: Float32Array.BYTES_PER_ELEMENT,
  onChunkProcessed: (float32Array) => {
    // Assign the received data to the pointsPositions array
    for (let i = 0; i < float32Array.length; i++) {
      pointsPositions[index++] = float32Array[i]
    }

    // Update points geometry for rendering during loading
    pointsGeometry.attributes.position.needsUpdate = true;
    invalidate();
  },
  onComplete: () => {
    // Render full model using the same position data
    // and hide points after loading is complete
    // ...
  }
});
```

![Simple loading](images/simple.gif#full "Simple loading")

### Performance

Models with high number of can be slow to render, especially since we are updating geometry position every time we receive a chunk. And more often than not, those models will have detailed triangles and points will be very close to each other, so it makes sense to just reduce the number of points to render.

How much reduction needed will depend on your use case so I'll leave it to you to decide. After having the final reduction ratio, we can store the points into a separate array and reserved the full array for the final model. The code will look like this:

```ts
  ...
  let index = 0;
  let reducedIndex = 0;

  for (let i = 0; i < float32Array.length; i++) {
    // Store full vertices for the final model

    // Keep only 1 in every *reductionRatio* vertices for rendering during loading
    if (index % 3 === 0 && Math.floor(index / 3) % reductionRatio === 0) {
      reducedPoints[reducedIndex++] = float32Array[i] ?? 0;
      reducedPoints[reducedIndex++] = float32Array[i + 1] ?? 0;
      reducedPoints[reducedIndex++] = float32Array[i + 2] ?? 0;
    }
    vertices[index++] = float32Array[i] ?? 0;
  }
  ...
```

### Advanced rendering

That's it for the simple approach. But if you want to take it a step further, you can implement your own shader to render the points. The limit here is your imagination. In my case, I used a shader to make points floating around to make them look "unsettled", and because I was also loading indexed geometry, triangles data will be loaded after vertices, so another layer of progress can be demonstrated by changing the color of points and stop them from moving gradually as the triangles are being loaded.

&nbsp;

![Advanced rendering](images/advanced.gif#full "Advanced rendering")

The following code block is an implementation of the custom shader, and please beware that I was using a z-up system instead of three.js's default y-up. This concludes the article, thank you for reading and I hope you can get something out of this long article.

```glsl
import * as THREE from "three";

const pointsVertexShader = /* glsl */ `
  uniform float uPointSize;
  uniform float uOpacity;
  uniform float uTrianglesProgress;
  uniform float uModelMinZ;
  uniform float uModelMaxZ;
  uniform float uTime;
  uniform float uFloatAmplitude;
  uniform float uZoom;
  
  varying float vNormalizedHeight;
  varying float vIsAtOrigin;
  varying float vIsAnimated;
  
  #include <common>
  
  // Pseudo-random function for varied movement
  float random(vec3 pos) {
    return fract(sin(dot(pos.xyz, vec3(12.9898, 78.233, 45.5432))) * 43758.5453);
  }
  
  void main() {
    // Start with original position
    vec3 pos = position;
    
    // Calculate normalized height (0-1) based on z position
    vNormalizedHeight = (position.z - uModelMinZ) / (uModelMaxZ - uModelMinZ);
    
    // Check if the point is at the origin (0, 0, 0)
    vIsAtOrigin = (position.x == 0.0 && position.y == 0.0 && position.z == 0.0) ? 1.0 : 0.0;
    
    // Determine if this point should be animated (above progress threshold)
    bool shouldAnimate = uTrianglesProgress == 0.0 || vNormalizedHeight > uTrianglesProgress;
    vIsAnimated = shouldAnimate ? 1.0 : 0.0;
    
    // Apply floating animation only to points above progress threshold
    if (shouldAnimate) {
      float uniqueOffset = random(position);
      float animSpeed = 1.0 + uniqueOffset * 0.8; // Varied speeds for different points
      
      // Create 3D floating motion with varied frequency
      float floatX = sin(uTime * animSpeed + uniqueOffset * 6.28) * uFloatAmplitude;
      float floatY = cos(uTime * animSpeed * 0.7 + uniqueOffset * 6.28) * uFloatAmplitude;
      float floatZ = sin(uTime * animSpeed * 0.5 + uniqueOffset * 6.28) * uFloatAmplitude * 0.7;
      
      pos += vec3(floatX, floatY, floatZ);
    }
    
    // Project with possibly modified position
    vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
    gl_Position = projectionMatrix * mvPosition;
    
    // Adjust point size based on zoom level
    // As zoom gets smaller (zooming out), we apply a scaling factor
    gl_PointSize = uPointSize * uZoom / 5.0;
  }
`;

const pointsFragmentShader = /* glsl */ `
  uniform vec3 uVerticesColor;
  uniform vec3 uTrianglesColor;
  uniform float uOpacity;
  uniform float uTrianglesProgress;
  
  varying float vNormalizedHeight;
  varying float vIsAtOrigin;
  varying float vIsAnimated;
  
  #include <common>
  #include <packing>
  
  void main() {
    // Initially all points are at origin, discard them to avoid artifacts
    if (vIsAtOrigin > 0.5) {
      discard;
    }

    // Create circular points instead of squares for better appearance
    vec2 cxy = 2.0 * gl_PointCoord - 1.0;
    float r = dot(cxy, cxy);
    if (r > 1.0) {
      discard;
    }
    
    float movingOpacity = uOpacity * 1.5;
    float trianglesOpacity = uOpacity * 6.0;
    
    bool useTrianglesColor = (uTrianglesProgress > 0.0) && (vNormalizedHeight <= uTrianglesProgress);
    
    vec3 finalColor = uVerticesColor;
    float finalOpacity = vIsAnimated > 0.5 ? movingOpacity : uOpacity;
    
    if (useTrianglesColor) {
      // Calculate the transition zone (below the progress threshold)
      float transitionZone = 0.05;
      float colorFactor = 0.4;
      float distanceFromProgress = uTrianglesProgress - vNormalizedHeight;
      
      if (distanceFromProgress <= transitionZone && distanceFromProgress >= 0.0) {
        // Create a darkening factor: 0.0 at start of transition zone, max at the progress line
        float darkFactor = colorFactor * (1.0 - (distanceFromProgress / transitionZone));
        // Apply darkening to triangles color
        finalColor = uTrianglesColor * (1.0 - darkFactor);
      } else {
        // Use regular triangles color
        finalColor = uTrianglesColor;
      }
      
      finalOpacity = trianglesOpacity;
    }
      
    gl_FragColor = vec4(finalColor, uOpacity);
  }
`;

export interface PointsShaderMaterialOptions {
  verticesColor?: THREE.Color;
  trianglesColor?: THREE.Color;
  opacity?: number;
  pointSize?: number;
  trianglesProgress?: number;
  modelMinZ?: number;
  modelMaxZ?: number;
  floatAmplitude?: number;
}

/**
 * Shader material for rendering points when loading
 * - Animates points above the progress threshold to float randomly
 */
export class PointsShaderMaterial extends THREE.ShaderMaterial {
  constructor(options: PointsShaderMaterialOptions = {}) {
    const {
      verticesColor = new THREE.Color(0xaaaaaa),
      trianglesColor = new THREE.Color(0x5555ff),
      opacity = 0.05,
      pointSize = 2.0,
      trianglesProgress = 0.0,
      modelMinZ = 0.0,
      modelMaxZ = 1.0,
      floatAmplitude = 0.1,
    } = options;

    super({
      vertexShader: pointsVertexShader,
      fragmentShader: pointsFragmentShader,
      uniforms: {
        uVerticesColor: { value: verticesColor },
        uTrianglesColor: { value: trianglesColor },
        uTrianglesProgress: { value: trianglesProgress },
        uOpacity: { value: opacity },
        uPointSize: { value: pointSize },
        uModelMinZ: { value: modelMinZ },
        uModelMaxZ: { value: modelMaxZ },
        uTime: { value: 0.0 },
        uFloatAmplitude: { value: floatAmplitude },
        uZoom: { value: 1.0 },
      },
      transparent: true,
    });
  }

  /**
   * Update the time uniform to animate the floating points
   */
  updateTime(time: number): void {
    this.uniforms.uTime.value = time;
  }

  /**
   * Set the float amplitude for the animation
   */
  setFloatAmplitude(amplitude: number): void {
    this.uniforms.uFloatAmplitude.value = amplitude;
  }

  /**
   * Set the z-range of the model for proper height normalization
   */
  setModelZRange(minZ: number, maxZ: number): void {
    this.uniforms.uModelMinZ.value = minZ;
    this.uniforms.uModelMaxZ.value = maxZ;
  }

  /**
   * Update the zoom level to adjust point size accordingly
   */
  updateZoom(zoom: number): void {
    this.uniforms.uZoom.value = zoom;
  }
}
```
