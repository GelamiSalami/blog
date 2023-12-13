---
title: "Raytracing Intersections: The Box"
date: 2023-12-04 23:45:00 +0800
categories: [Blog]
tags: [raytracing, ray-intersection, box]
hidden: true
math: true
---

[Previously](/blog/posts/ray-plane-intersection/), we've discussed how to intersect a ray with a plane. We'll be extending what we've learned to intersect boxes.

Recall that the ray intersection of a plane, with normal $ \hat N $ that's centered at the position $ P $, can be defined as:

$$ t = -\frac{(O - P) \cdot \hat N}{\hat D \cdot \hat N} $$

As mentioned previously, if the normal $ \hat N $ is axis-aligned, e.g. $ \hat N = (1, 0, 0) $, the equation would simplify to the scalar components on the normal's axis:

$$ t_x = -\frac{O_x - P_x}{\hat D_x} $$

Likewise for $ N = (0, 1, 0) $ and $ N = (0, 0, 1) $:

$$ t_y = -\frac{O_y - P_y}{\hat D_y} $$

$$ t_z = -\frac{O_z - P_z}{\hat D_z} $$

We can combine this to a single vector $ T_{xyz} $ which we use to compute the intersection of the three axis-aligned planes simultaneously:

$$ T_{xyz} = -\frac{O - P}{\hat D} $$

## Derivation

### With min and max positions

Let's define an axis-aligned bounding box (AABB) as two vectors of its corners, the minimum $$ B_{min} $$ and the maximum $$ B_{max} $$ positions.

We can take the plane intersections on each axis at those positions, which gives us the intersection for each plane of the box:

$$ T_{min} = -\frac{O - B_{min}}{\hat D} $$

$$ T_{max} = -\frac{O - B_{max}}{\hat D} $$

To check if the ray intersects the box, we need to separate the planes as either a near or a far intersection, where the near intersection is the plane closer to the ray origin, and the far intersection the farther at the opposite.

We can get these by taking the minimum and the maximum respectively on a pair of parallel planes:

$$ x_{near} = \min(x_{min}, x_{max}) $$

$$ x_{far} = \max(x_{min}, x_{max}) $$

Applying it to our set of planes, we get the near and far intersections for each axis:

$$ T_{near} = \min(T_{min}, T_{max}) $$

$$ T_{far} = \max(T_{min}, T_{max}) $$

We combine to get the final intersection: // TODO Clarify

$$ t_{near} = \max(\max(T_{near_x}, T_{near_y}), T_{near_z})$$

$$ t_{far} = \min(\min(T_{far_x}, T_{far_y}), T_{far_z})$$

Finally, we can detect a hit if the near distance is less than the far distance:

$$ t_{near} < t_{far} $$

### With center and half-size

We define the box as it's center $ P $ and its half if its size as $ S $

Recall the plane intersection formula with the plane offset $ d $:

$$ t = -\frac{O \cdot \hat N - d}{\hat D \cdot \hat N} $$

This offsets the plane toward the normal $ \hat N $. But what if we want to offset it away from us at any view? We can do so by multiplying $ d $ by the sign of the denominator: // TODO elaborate

$$ \DeclareMathOperator{\sign}{sign} $$

$$ t_{far} = -\frac{O \cdot \hat N - d * \sign(\hat D \cdot \hat N)}{\hat D \cdot \hat N} $$

Likewise, for a plane offset toward us, we take the opposite sign:

$$ t_{near} = -\frac{O \cdot \hat N + d * \sign(\hat D \cdot \hat N)}{\hat D \cdot \hat N} $$

We can rewrite the above formula, by separating the offset term and substituting $ \frac{sign(x)}{x} $ for $ \frac{1}{abs(x)} $, we get:

$$ t_{near} = -\frac{O \cdot \hat N}{\hat D \cdot \hat N} + \frac{d}{abs(\hat D \cdot \hat N)} $$

$$ t_{far} = -\frac{O \cdot \hat N}{\hat D \cdot \hat N} - \frac{d}{abs(\hat D \cdot \hat N)} $$

Applying that to our three axis-aligned planes, we get our formula:

$$ T_{near} = -\frac{O - P}{\hat D} + \frac{S}{abs(\hat D)} $$

$$ T_{far} = -\frac{O - P}{\hat D} - \frac{S}{abs(\hat D)} $$

Finally, we get our near and far distances as before:

$$ t_{near} = \max(\max(T_{near_x}, T_{near_y}), T_{near_z})$$

$$ t_{far} = \min(\min(T_{far_x}, T_{far_y}), T_{far_z})$$

## Finding the normal

TODO

$$ f(x)= \begin{cases}
    (1, 0, 0), & \text{if } T_{near_{x}} > T_{near_{z}} \text{ and } T_{near_{x}} > T_{near_{z}} \\
    (0, 1, 0), & \text{else if } T_{near_{y}} > T_{near_{z}} \\
    (0, 0, 1)  & \text{else}
\end{cases} $$

$$ \hat N_{box} = f(x) \sign(\hat D) $$

##  Oriented Bounding Box

For oriented bounding boxes, we transform the ray origin and direction using the oriented bounding box's rotation matrix $ M $:

$$ O_{M} = M * O $$

$$ D_{M} = M * D $$

TODO

## In code

Min-max extents

```glsl
float rayBoxIntersection(vec3 rayOrigin, vec3 rayDir, vec3 aabbMin, vec3 aabbMax) {
    vec3 planesMin = -(rayOrigin - aabbMin) / rayDir;
    vec3 planesMax = -(rayOrigin - aabbMax) / rayDir;

    vec3 planesNear = min(planesMin, planesMax);
    vec3 planesFar  = max(planesMin, planesMax);

    float tNear = max(max(planesNear.x, planesNear.y), planesNear.z);
    float tFar  = min(min(planesFar.x, planesFar.y), planesFar.z);

    if (tNear > tFar) {
        return MAX_DISTANCE; // No hit
    }
    return tNear;
}
```

Center and half-size

```glsl
float rayBoxIntersection(vec3 rayOrigin, vec3 rayDir, vec3 center, vec3 halfSize) {

    vec3 planeHit = (rayOrigin - center) / rayDir;
    vec3 planeOffset = halfSize / abs(rayDir);

    vec3 planesNear = -planeHit + planeOffset;
    vec3 planesFar  = -planeHit - planeOffset;

    float tNear = max(max(planesNear.x, planesNear.y), planesNear.z);
    float tFar  = min(min(planesFar.x, planesFar.y), planesFar.z);

    if (tNear > tFar) {
        return MAX_DISTANCE; // No hit
    }
    return tNear;
}
```

Normal

```glsl
vec3 mask;
if (planesNear.x > planesNear.y && planesNear.x > planesNear.z) {
    mask = vec3(1, 0, 0);
} else if (planesNear.y > planesNear.z) {
    mask = vec3(0, 1, 0);
} else {
    mask = vec3(0, 0, 1);
}

vec3 normal = -sign(rayDir) * mask;
```

In the next post, we will be talking about traversing a voxel grid. Seeya! 🐸