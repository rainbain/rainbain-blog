---
title: GPU Rasterizer Optimizations, Shared Exponents
date: 2026-10-11
draft: false
---
# GPU Rasterizer Optimizations, Shared Exponents
![[thumbnail.png]]
## Project Context
I have spent quite some time designing my own versions of GPUs and reverse engineering other GPU pipelines and hardware modules. Despite this, I have not yet posted on my GPU work here.

I have been developing a small GPU, inspired by tile based deferred renderers small enough to fit on the ICE40UP5K FPGA inside the phone I made. Furthermore, after that, I am hoping to tape it out on waferspace.

Due to these constraints, the GPU has to be as small and simple as possible, while still being capable. So I have had to come up with all sorts of optimizations.

After getting this optimization working on my own rasterizer, I wanted to bring it up as I often forget about using shared exponents. Hopefully I at some point can also write an introduction to rasterizers to better explain some of the core concepts and "whys" here, but for now this will serve as guidance to anyone possibly developing a GPU and as a strong example use case of where shared exponents can shine.

## The Problem
The fragments must interpolate vertex attributes using barycentric coordinates before using them. However, the rasterizer does not usually output barycentric coordinates directly. It instead outputs the 3 values of the edge function. Then it divides them by the triangle's area. After this we get our standard 3 component vector with values ranging `[0, 1]` for how close each pixel is to each vertex of the triangle.

This division per pixel is pretty wasteful, so usually I will have the processor before the rasterizer include the reciprocal area so it can just carry out a multiplication and be done.

But the issue here is precision if you choose to use fixed point. Small fixed function GPUs, and especially those on an FPGA, can quickly get bloated with floating point normalization hardware. So it's generally a good idea to try and avoid them for small low power GPUs.

But this multiplication, when represented as fixed point, is problematic. Assume a max framebuffer resolution of 640x480. The possible range of triangle areas, assuming they're clipped at exactly the screen borders, is `153600` pixels.

So now, both the area and edge values have a range of `1-153600`. Worse, if we choose to send a reciprocal area, we will ideally want a good few extra fractional bits so that small area triangles are not just divided by `1`.

So we end up with what tends to become a `32x32` multiplication. But do we really need all that precision for our barycentric coords? If your GPU is like my small tile based system, texture resolution is so limited that you only need a `Q0:8` output. So it would be ideal if we could just use an `8x8` multiplication in the fragments. This would make it possible for it to only ever use 1 fixed point format and remove a lot of complexities as it would be only using 1 format in its multipliers.

Attempting to solve this problem has a lot of pitfalls that I will list at the end, and why I have found this solution to the problem to be the most painless.

## Developing the Solution
There are some very important relationships between the edge functions and the triangle's area:
### Value Ranges
Since dividing the edge functions by the triangle area yields outputs ranged from `[0, 1]`, we know that the range for the edge functions output must be `[0, area]`.

### Barycentric Coords are a Directional Quantity
Barycentric coords tell us where on the surface of the triangle we are. They are a direction, for where we are on the triangle.

This is very different from a position in the triangle per se. Although they're not unit length, we generally care more about the "direction" the coordinates face in than their exact position.

### Meet Shared Exponent Floats
Shared exponent floats fit the bill here really well. This is when a vector's components all share the same exponent.

This works well on directional vectors since small offsets will be quantized to zero. You for example would not want to use this to represent the position of an object in a game world. It would lose precision on the X and Y axes as it ventured farther on the Z axis resulting in unintuitive loss in precision.

But for things like colors and rotations this is fine. We care more about the ratios between the values. For example modern GPUs let you use `RGB9_E5`. Where 3 color channels share the same exponent. Now you can store high dynamic range colors in only 32 bits instead of 96 bits, by sharing 1 exponent.

### Apply to Rasterizer
So we can see why we could express barycentric coordinates this way, but how does this help us?

Let's think about how we could turn that `32x32` multiplication into a `32x16` multiplication. We keep the value of the edge function as 32 bits. Well as a triangle gets big, the reciprocal area gets small. So we could switch between two exponent values. You do need to leave some overlap between the two sections, so that when you switch, you don't suddenly drop a bunch of precision between the boundaries. That looks like this:
```c
// Saturated Q0:32 reciprocal
uint64_t area_recip = (double)(0x100000000) / area;
if(area_recip > 0xFFFFFFFF)
    area_recip = 0xFFFFFFFF;

int shift;
uint16_t s_area_recip;
if(area_recip > 0x00FFFFFF) {      // area < 256
    shift = 16;
    s_area_recip = area_recip >> 16;
} else {                           // area >= 256
    shift = 8;
    s_area_recip = area_recip >> 8;
}

...
// Apply to Edge Function
int32_t f_w0 = (w0 * s_area_recip) << shift; // ~ (w0 / area) * 2^32
```

That works ok, but the precision is weirdly variable, and I'd like to do an `8x8` multiplication. So we can expand this.

Instead of switching between two fixed point formats, I want to guarantee that after multiplying by the reciprocal area that the product is always the same format. That's pretty easy actually.

We can just say that the reciprocal area is `Q0:8` and the edge function is `Q0:8`, and the product will always be `Q0:16`. But how do we force these to the same format? Shared exponents!

And here is the lovely thing, we already know the optimum exponent to use. The edge functions are ranged by the area. So the exponent should be large enough to fit the triangles area. So whatever 8 bit window we would use for the area, will be the same for the edge function.

### Implementation
#### Priority Encoder
First, let's make some helper functions. I will call this, select the window and grab it, the priority encoder. Like a floating point normalization, it's truncating the number and giving us an exponent.

```c
// Get the index of the first set most significant bit
static int priority_bit(int32_t b) {
	for(int i = 31; i >= 0; i--) {
		if(b & (1<<i))
			return i;
	}
	return 0;
}

  
// Get an 8 bit window of that bit and the 7 bits following it
static uint8_t priority_encode(int32_t in, int bit) {
	int shift = bit - 7;
	if(shift > 0) {
		return in >> shift;
	} else {
		return in << (-shift);
    }
}
```

#### Encoding the Area
```c
// Calculate area using edge function
int area = dw1_dy * dw2_dx - dw1_dx * dw2_dy;

// Flip triangles with negative area
if (area < 0) {
	area = -area;
	dw0_dx = -dw0_dx; dw0_dy = -dw0_dy;
	dw1_dx = -dw1_dx; dw1_dy = -dw1_dy;
	dw2_dx = -dw2_dx; dw2_dy = -dw2_dy;
}

// Exponent we will use for the edge values
int area_exp = priority_bit(area);

// Saturated Q0:32 reciprocal
uint64_t area_recip = (double)(0x100000000) / area;
if(area_recip > 0x100000000)
	area_recip = 0x100000000;

// Encode the reciprocal
int recip_exp = priority_bit(area_recip);

// 1 minus for an implied leading 1 removed for free extra precision
uint8_t area_recip_enc = priority_encode(area_recip, recip_exp-1);
```

#### Encoding Edge Values
For what would be done by the rasterizer per pixel.
```c
uint8_t w0_enc = priority_encode(w0, area_exp);
uint8_t w1_enc = priority_encode(w1, area_exp);
uint8_t w2_enc = priority_encode(w2, area_exp);
```

#### Generating Barycentrics
And now for the multiplication the fragments would do to yield 3 barycentric coords.
```c
// Add w0_enc to account for the implied 1 in area_recip_enc.
uint8_t b0 = ((w0_enc * area_recip_enc) >> 8) + w0_enc;
uint8_t b1 = ((w1_enc * area_recip_enc) >> 8) + w1_enc;
uint8_t b2 = ((w2_enc * area_recip_enc) >> 8) + w2_enc;
```

#### Full Rasterizer + Colored Triangle
```c
// Get the index of the first set most significant bit
static int priority_bit(int32_t b) {
	for(int i = 31; i >= 0; i--) {
		if(b & (1<<i))
			return i;
	}
	return 0;
}

  
// Get an 8 bit window of that bit and the 7 bits following it
static uint8_t priority_encode(int32_t in, int bit) {
	int shift = bit - 7;
	if(shift > 0) {
		return in >> shift;
	} else {
		return in << (-shift);
    }
}

static uint16_t rgb565(uint8_t r, uint8_t g, uint8_t b) {
	return (uint16_t)(((r & 0xF8) << 8) | ((g & 0xFC) << 3) | (b >> 3));
}

static void rasterize(uint16_t tile[TILE][TILE],
               int ax, int ay,
               int bx, int by,
               int cx, int cy) {
    // Compute bounding box by accumulating MIN/MAXes, clipped to the tile
    int16_t minx = MAX(MIN(MIN(ax, bx), cx), 0);
    int16_t miny = MAX(MIN(MIN(ay, by), cy), 0);
    int16_t maxx = MIN(MAX(MAX(ax, bx), cx), TILE - 1);
    int16_t maxy = MIN(MAX(MAX(ay, by), cy), TILE - 1);

    // Triangle doesn't touch this tile
    if (minx > maxx || miny > maxy)
        return;

    // w0 derivatives
    int16_t dw0_dx =  cy - by;
    int16_t dw0_dy =  bx - cx;

    // w1 derivatives
    int16_t dw1_dx =  ay - cy;
    int16_t dw1_dy =  cx - ax;

    // w2 derivatives
    int16_t dw2_dx =  by - ay;
    int16_t dw2_dy =  ax - bx;

    // Differences (evaluated at the clipped bbox corner, so the
    // incremental walk starts at the tile's first covered pixel)
    int16_t minx_ax = minx - ax;
    int16_t minx_bx = minx - bx;
    int16_t minx_cx = minx - cx;

    int16_t miny_ay = miny - ay;
    int16_t miny_by = miny - by;
    int16_t miny_cy = miny - cy;

    // Calculate area using edge function
    int area = dw1_dy * dw2_dx - dw1_dx * dw2_dy;

    if (area < 0) {
        area   = -area;
        dw0_dx = -dw0_dx;  dw0_dy = -dw0_dy;
        dw1_dx = -dw1_dx;  dw1_dy = -dw1_dy;
        dw2_dx = -dw2_dx;  dw2_dy = -dw2_dy;
    }

    // Exponent we will use for the edge values
    int area_exp = priority_bit(area);

    // Saturated Q0:32 reciprocal
    uint64_t area_recip = (double)(0x100000000) / area;
    if(area_recip > 0x100000000)
        area_recip = 0x100000000;

    // Encode the reciprocal
    int recip_exp = priority_bit(area_recip);

    // 1 minus for an implied leading 1 removed for free extra precision
    uint8_t area_recip_enc = priority_encode(area_recip, recip_exp-1);

    
    // Compute initial edge functions
    int32_t w0_i = minx_bx * dw0_dx + miny_by * dw0_dy;
    int32_t w1_i = minx_cx * dw1_dx + miny_cy * dw1_dy;
    int32_t w2_i = minx_ax * dw2_dx + miny_ay * dw2_dy;

    int32_t w0 = w0_i;
    int32_t w1 = w1_i;
    int32_t w2 = w2_i;

    for(int y = miny; y <= maxy; y++) {
        w0 = w0_i;
        w1 = w1_i;
        w2 = w2_i;
        for(int x = minx; x <= maxx; x++) {
            // If in triangle
            if((w0 >= 0 && w1 >= 0 && w2 >= 0)) {

                uint8_t w0_enc = priority_encode(w0, area_exp);
                uint8_t w1_enc = priority_encode(w1, area_exp);
                uint8_t w2_enc = priority_encode(w2, area_exp);

                // Add w0_enc to account for the implied 1 in area_recip_enc.
                uint8_t b0 = ((w0_enc * area_recip_enc) >> 8) + w0_enc;
                uint8_t b1 = ((w1_enc * area_recip_enc) >> 8) + w1_enc;
                uint8_t b2 = ((w2_enc * area_recip_enc) >> 8) + w2_enc;

                tile[x][y] = rgb565(b0, b1, b2);
            }

            w0 += dw0_dx;
            w1 += dw1_dx;
            w2 += dw2_dx;
        }

        w0_i += dw0_dy;
        w1_i += dw1_dy;
        w2_i += dw2_dy;
    }
}
```

## Results
Results are good but not perfect.

### Big Triangles
Big triangles come out perfect.
![[content/posts/gpu_rasterizer_optimizations_shared_exponents/big.png]]

### Small Triangles
These also come out very good.
![[content/posts/gpu_rasterizer_optimizations_shared_exponents/small.png]]

#### Weird And Small Triangles
But certain small triangles seem to throw it off. You can see by the darkening here there was a good bit of precision loss. It's rather random when it does this. At Q0:8 I suspect there are cases where 1 bit of precision gets lost. A more reasonable Q0:16 would likely fix this.
![[content/posts/gpu_rasterizer_optimizations_shared_exponents/weird.png]]

### Skinny Triangles
All good!
![[skinny.png]]

### Texturing Demo
![[texturing.png]]
For the most part good. We can see the Q0:8 format and my texture sampler don't get along fully right now in the software renderer though as evidenced by seam bleeding. But this is commonplace for low precision UVs.

## Common Pitfalls
Now I will list the pitfalls I fell into before finding this solution to help some developers maybe avoid them. They may have their own tricks to overcome that I have not found.

### Normalize the Edge Functions First
You can modify the deltas such that the produced edge values always make the output fixed point format we want. I mainly avoided this because it caused holes in models. The deltas measured by pixels avoided a lot of rounding errors.
Without it, I'd often get the vertices of the triangle wiggling around.

It also made those delta values big precision, as for small triangles it's a big value, and for big ones it's a small value. That resulted in big `32+32` adders, and big `32x32` multipliers in my vertex setup hardware I did not like.
### Add a Separate Barycentric Coord Function.
Like we have the deltas for edge functions, you could add a normalized delta for Barycentric Coords. It suffers from the first issue in terms of requiring a lot more setup, and a lot more data sent to the rasterizer per triangle.

I am operating a deferred tile based renderer, so it's best to minimize triangle bandwidth at this stage so I also avoided this solution.

With that many counters it's also just lighter to do the whole shared exponent trick.

## Conclusion
I am very happy with this outcome and will likely continue to use it on my future GPUs. It means the fragments can pretty much just use the Barycentric Coords as is, and if you ever decided to move to floats, the conversion is pretty much free! All while the rasterizer is fast and simple.

I have probably spent a month trying to optimize my rasterizer in weird ways, and have been so happy to find this elegant solution.
