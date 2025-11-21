# 2D LUT

This article created to be starting point for a proposal to standardize 2D LUT format for color grading applications.

## Motivation
Today there is no standard and efficient way to describe color transforms in gamut space. 

Examples of such transforms are:
 - Gamut compression
 - Gamut mapping of any kind. Mostly used to convert camera responses to an aproximation of human vision stimuli
 - Linear Gamut transform = Matrix3x3 multiplication.

Please refer to the [CinemaCameraBeautifiers](../CinemaCameraBeautifiers) article for more details about such transforms.

## What 2D LUT does
2D LUT deforms this triangle:
{% include 3DLUTViewer showChromaTriangle=16 %}
{% include Gap %}
Every line from 0 (black) to any color will be deformed into a straight line from 0 (black) to the deformed color, preserving linearity along this line.
This important property will be explained later in the article.

## Existing ways to describe such transforms
Existing way to describe such transforms is to use 3D LUTs. But 3D LUTs are suboptimal for this purpose, because:

- Memory consumption. If we need 512 steps between, let's say, Red and Green, we need **3D LUT** of size 513x513x513 = 135005697 entries. Each entry is 3 floats (RGB), so total size is **1582 MB** in binary format. In contrast, **2D LUT** with 512 steps between Red and Green needs only 513x256 = 131,328 entries = **1.5 MB** in binary format.

- Resolution. 3D LUT of size 33x33x33 has 32 steps between 100% Red and 100% Green, but **if brightness decreases**, for example to 25% Red and 25% Green, the number of steps between these colors decreases to only 8 steps. In contrast, 2D LUT of size 33 always has 32 steps, no matter what brightness is.

## Coordinate mapping
In order to apply 2D LUT to input 3D color, we need to map input color to 2D coordinates. Here is the proposed mapping: 
``` hlsl
// project RGB color to a plane x+y+z=1
float sum = inputColor.r + inputColor.g + inputColor.b;

if (sum < EPSILON)
    return float3(0,0,0); // black

float2 uv = inputColor.rg / sum;

float3 transformedColor = Sample2DLUT(uv);

transformedColor *= sum; // restore scale

return transformedColor;
```

For further explanation, let's use LUT with 6 nodes along the edge (including corners).

## Mapping to texture pixels
Out goal is to store 2D LUT data in GPU-memory as a texture and sample it in shader code. 
The LUT discribes a triangular lattice; texture is rectangular. So we need to define how to map 2D LUT nodes to texture pixels.
We can shift each row to the left to align nodes with texture pixels:
{% include_relative Example.svg commands="DrawLattice();AnimateTrianglesHorizontalShift();" %}
{% include Gap %}


## Packing
As you may notice, half of the texture pixels are unused. We can pack the triangle. Let's cut the upper half and put it to the right side:

{% include_relative Example.svg commands="AnimateUpperTriangle();" %}
{% include Gap %}

## Sampling
To sample 2D LUT, we can not use bilinear filtering, because we stored a triangular lattice. Instead, we need to find the triangle where the input coordinates are located, load that 3 values and use barycentric coordinates to interpolate between them.

Here is the HLSL code to sample Packed 2D LUT:
``` hlsl
//LUT is a Texture2D<float4> containing 2D LUT data
float3 Sample2DLUT(float2 uv) {
    uint sizeX;
    uint sizeY;
    LUT.GetDimensions(sizeX, sizeY);

    float2 pixelPosition = uv * float2(sizeX - 2, 2 * sizeY - 1);

    int2 pixelPosition00 = int2(pixelPosition);
    //pixelPosition00 is a position of rectangle we are in

    int2 pixelPosition10 = pixelPosition00 + int2(1, 0);
    int2 pixelPosition01 = pixelPosition00 + int2(0, 1);

    float2 frac = pixelPosition - pixelPosition00;
    //rectangle is made of two triangles, determine which one we are in
    if (frac.x + frac.y > 1) { //upper triangle
        frac = (1 - frac).yx;
        pixelPosition00 += int2(1,1);
    }

    //handle cutted corner of a triangle
    if (pixelPosition00.y >= sizeY) {
        pixelPosition00 = int2(sizeX-1 ,2*sizeY-1) - pixelPosition00;
    }
    if (pixelPosition10.y >= sizeY) {
        pixelPosition10 = int2(sizeX-1 ,2*sizeY-1) - pixelPosition10;
    }
    if (pixelPosition01.y >= sizeY) {
        pixelPosition01 = int2(sizeX-1 ,2*sizeY-1) - pixelPosition01;
    }

    //load 3 LUT values
    float3 transformed00 = LUT.Load(int3(pixelPosition00, 0)).rgb;
    float3 transformed10 = LUT.Load(int3(pixelPosition10, 0)).rgb;
    float3 transformed01 = LUT.Load(int3(pixelPosition01, 0)).rgb;

    //interpolate using barycentric coordinates
    float3 result =
        transformed00
        + frac.x * (transformed10 - transformed00)
        + frac.y * (transformed01 - transformed00);

    return result;
}
```

## Image format
2D LUT can be stored in an image file with lossless compression and float data support, for example, EXR;

For 2D LUT with N nodes along the edge, image size should be:
- For Packed version:
    - Width = N + 1
    - Height = N / 2
    
Packed version supports only even N.


## Text format
To be similar to existing [3D LUT text format](https://kono.phpage.fr/images/a/a1/Adobe-cube-lut-specification-1.0.pdf), here is proposed text format for 2D LUT:

```
# 2D LUT    
LUT_2D_SIZE N
TITLE "Optional title"
DOMAIN_R 1.0 0.0 0.0
DOMAIN_G 0.0 1.0 0.0
DOMAIN_B 0.0 0.0 1.0
r0 g0 b0
r1 g1 b1
...

rM-1 gM-1 bM-1
```
Where `N` = number of nodes along the edge, `M = (N+1) * N / 2` = total number of nodes.

## Ordering of nodes in 2D LUT
{% include_relative Example.svg commands="AddIndices();" %}
{% include Gap %}
In 2D LUT of size 6
Node with index **0** contanins new value for input color (0,0,1) = **Blue**. If input color is scaled Blue, for example (0,0,0.5), output color will be multiplied by 0.5 as well.
Node with index **5** contains new value for input color (1,0,0) = **Red**. 
Node with index **20** contains new value for input color (0,1,0) = **Green**.


## Domain
By default, we project the input color onto the triangle defined by the points (1,0,0), (0,1,0), and (0,0,1).
However, if we design a LUT to describe a transformation of a specific portion of the color space or the opposite, to handle out-of-gamut colors (input colors with negative components), we can define a custom domain using the `DOMAIN_R`, `DOMAIN_G`, and `DOMAIN_B` tags in the text format.

`DOMAIN_R`, `DOMAIN_G`, and `DOMAIN_B` are vectors in input space that define the corners of the triangle onto which input colors will be projected before sampling the 2D LUT. 
Using these vectors, we can build a 3x3 matrix
`DomainToInputMatrix = [DOMAIN_R; DOMAIN_G; DOMAIN_B]`. This matrix is a thansform from internal domain space to input space.

Next, we calculate the inverse matrix `InputToDomainMatrix = inverse(DomainToInputMatrix)`. This matrix transforms input colors to the DOMAIN space and should be used in the shader code before projecting the color onto the triangle.


## Conclusion
I truly believe that this kind of image transform is essential in cinematography and virtual production workflows. For most cases, a combination of 1D and 2D LUTs can replace 3D LUTs with higher precision and lower memory consumption, potentially leading to more GPU-cache-friendly algorithms.
