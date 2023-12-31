---
title: "Raytracing Intersections: Voxel Traversal"
date: 2023-12-21 01:55:00 +0800
categories: [Blog]
tags: [raytracing, ray-intersection, box, voxel, traversal]
hidden: true
math: true
---

Previously, we've derived the ray intersection of a box. Today, we are going to use that to traverse a voxel grid.

We can think of traversing a voxel grid as doing a series of box intersections, stepping from one box to the next.

// Show diagram of voxel traversal

First, we find the starting voxel position, which is the voxel that our ray origin is in.
Then we intersect the inside of a box at that position, which gives us a normal which we use to step to the next voxel position. We repeat this until we find a solid voxel.

So our voxel traversal function would be like such:
- Find the voxel position of the ray
- While a solid voxel is not hit
    - Intersect voxel to find the axis to step through
    - Update voxel position using the step axis


We can find the voxel position from our ray origin `rayOrig` by taking its integer coordinates using `floor`:

```glsl
vec3 voxelPos = floor(rayOrig);
```

We intersect the voxel by doing a ray-box intersection (as we've discussed in [the previous post](/blog/posts/ray-box-intersection/)), but we only take the back-facing intersections since the ray is always inside the voxel.

We also add `0.5` to the voxel position `voxelPos` to get the center position of the voxel:

```glsl
vec3 planeDist = (rayOrig - (voxelPos + 0.5)) / rayDir;
vec3 planeOffset = 0.5 / abs(rayDir);
vec3 sideDist = -planeDist + planeOffset;
```

For brevity, we can combine this into a single line:

```glsl
vec3 sideDist = -(rayOrig - (voxelPos + 0.5)) / rayDir + 0.5 / abs(rayDir);
```

To find the step direction, we find the axis of the smallest intersection, this is equivalent to the normal of the box but pointing oppositely:

```glsl
vec3 mask = sideDist.x < sideDist.y && sideDist.x < sideDist.z ? vec3(1, 0, 0) :
            sideDist.y < sideDist.z ? vec3(0, 1, 0) : vec3(0, 0, 1);

vec3 stepDir = mask * sign(rayDir);
```

We add that to our voxel position `voxelPos` to move to the next voxel:
```glsl
voxelPos += stepDir;
```

Our traversal function would now look something like this:

```glsl
// Get voxel position
vec3 voxelPos = floor(rayOrig);

for (int i = 0; i < MAX_STEPS; i++) {
    if (isSolid(voxelPos)) {
        // Hit
        return;
    }

    // Intersect voxel
    vec3 sideDist = -(rayOrig - (voxelPos + 0.5)) / rayDir + 0.5 / abs(rayDir);

    // Find step direction
    vec3 mask = sideDist.x < sideDist.y && sideDist.x < sideDist.z ? vec3(1, 0, 0) :
                sideDist.y < sideDist.z ? vec3(0, 1, 0) : vec3(0, 0, 1);
    vec3 stepDir = mask * sign(rayDir);

    // Step to next voxel
    voxelPos += stepDir;
}
```

We can simplify this a bit by taking the terms that doesn't change and precomputing them, such as the related terms from the ray direction:

```glsl
vec3 invRayDir = 1.0 / rayDir;
vec3 deltaDist = abs(invRayDir);
vec3 raySign = sign(rayDir);
```

Another simplification can be done with the side distance `sideDist`.
An observation can be made that the only term that changes for each iteration is the voxel position `voxelPos`, which is offset by the step direction `stepDir`.

So instead of computing the `sideDist` every step, we can precompute it before the loop and add the remainder for every step instead:

```glsl
// Before the loop
vec3 sideDist = -(rayOrig - (voxelPos + 0.5)) * invRayDir + 0.5 * deltaDist;
```
```glsl
// In the loop
sideDist += mask * deltaDist;
```

Our traversal function would now look like this:

```glsl
vec3 invRayDir = 1.0 / rayDir;
vec3 deltaDist = abs(invRayDir);
vec3 raySign = sign(rayDir);

vec3 voxelPos = floor(rayOrig);

vec3 sideDist = -(rayOrig - (voxelPos + 0.5)) * invRayDir + 0.5 * deltaDist;

for (int i = 0; i < MAX_STEPS; i++) {
    if (isSolid(voxelPos)) {
        // Hit
        return;
    }
    
    vec3 mask = sideDist.x < sideDist.y && sideDist.x < sideDist.z ? vec3(1, 0, 0) :
                sideDist.y < sideDist.z ? vec3(0, 1, 0) : vec3(0, 0, 1);
    
    voxelPos += mask * raySign;
    sideDist += mask * deltaDist;
}
```

When a hit occurs, we need to return information about the voxel, such as the voxel position, normal, and the hit distance.

The voxel position is simply `voxelPos`.

We can get the normal by storing the previous `mask` then multiplied by the sign of the ray direction:

```glsl
normal = -mask * raySign;
```

The hit distance `t` can be taken from the minimum value of the side distance `sideDist` from the previous iteration, so we undo the addition before:

```glsl
vec3 prevSideDist = sideDist - mask * deltaDist;
t = min(min(prevSideDist.x, prevSideDist.y), prevSideDist.z);
```

This concludes this post on the traversal of a voxel grid. Some improvements on this would include brickmaps, octrees, custom intersections in the voxel, etc., perhaps we can discuss it next time.

Next post we will be talking about the generalized ray-intersection of convex polyhedra. Now go traverse your voxel worlds! Seeya! 🐸