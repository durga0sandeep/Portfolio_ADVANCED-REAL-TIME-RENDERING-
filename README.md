# CS562 – Advanced Real-Time Rendering

A semester-long real-time rendering engine built. Each project builds on the previous one, starting from a deferred renderer and adding filtered soft shadows, physically based and image-based lighting, and screen-space ambient occlusion.


### Project 1: Deferred Shading
This project converts a forward renderer into a **deferred shading** pipeline, so the scene can be lit by many local lights efficiently.

- **G-buffer pass:** The scene is rendered into a multiple-render-target FBO that stores world position, surface normal, diffuse color, and specular properties. No lighting is computed in this pass.
- **Lighting pass:** A full-screen quad reads the G-buffer textures and computes lighting (N, L, V vectors) for every pixel in a single pass over the image.
- **Many local lights pass:** Each local light is drawn as a bounding sphere around its range. The contributions are accumulated with additive blending.
- **Smooth attenuation:** The falloff is `1/d² − 1/r²`. It approximates inverse-square falloff but reaches exactly zero at the light's range, so there is no hard edge.
- **Shadow pass:** The existing shadow-map pass (from previous class project) is kept and runs before the lighting passes.
- **Debug views:** The diffuse color, normal, world position, and specular G-buffer channels can each be displayed for verification.
  
![Image 1](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj1_0.png)
![Image 2](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj1_1.png)

---


### Project 2 – Moment Shadow Mapping (MSM)

This project replaces the basic binary shadow-map test with **Moment Shadow Maps**, which produce filterable, anti-aliased soft shadows.

- The shadow map stores four depth moments `(z, z², z³, z⁴)` instead of a single depth. Depths are normalized to a relative `[0, 1]` range.
- A **separable Gaussian convolution blur** filters the moment map between generation and use. It runs as two compute shaders: one blurs horizontally, the other blurs the result vertically. The kernel width is adjustable. Results are shown at kernel sizes 2, 10, and 20, from slightly anti-aliased edges to very soft, wide-light shadows.
- **Variance Shadow Mapping** uses the first two moments with Chebyshev's inequality: `s = σ² / (σ² + (t − μ)²)`.
- The full **Hamburger 4MSM** algorithm solves a 3×3 linear system from the blurred moments, finds the roots of a quadratic, and returns a continuous shadow intensity `G ∈ [0, 1]`. This reduces the light leaking seen with VSM.
- **Comparisons:** The basic shadow map, VSM, and MSM can be rendered side by side, along with the depth map before and after blurring.

**Reference:** C. Peters & R. Klein, *Moment Shadow Mapping*, I3D 2015. [Paper](https://momentsingraphics.de/Media/I3D2015/MomentShadowMapping.pdf)

![Image 3](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj2_0.png)
![Image 4](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj2_1.png)
![Image 5](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj2_2.png)

---

### Project 3 – Physically Based Shading, Image-Based Lighting & Irradiance Maps

This project replaces the constant ambient term with **image-based lighting** from an HDR environment and adds a physically based micro-facet BRDF.

- **HDR environment:** An `.hdr` image is loaded with `stbi_loadf` into a floating-point OpenGL texture (`GL_FLOAT`) and wrapped around a skydome. The skydome supplies directional ambient light.
- **Irradiance map (CPU, multithreaded):** For each direction on the sphere, the diffuse irradiance is computed by numerically integrating the incoming light, using the clamped dot product, `sin θ` weighting, and angular step sizes `Δθ`, `Δφ`. The result is saved as a low-resolution HDR irradiance map.
- **Diffuse IBL:** `Kd/π · irradiance(N)`, a single irradiance-map lookup in the normal direction.
- **Specular IBL:** The specular integral is estimated with **Monte-Carlo importance sampling**. N directions are chosen according to the BRDF's probability distribution, the HDR environment is read in each direction, and the estimator is summed and averaged.
- **Tone mapping & color space:** All lighting is computed in linear color space. Tone mapping compresses HDR values to the displayable range, and the result is converted to sRGB. Exposure and gamma can be adjusted interactively.
- **Comparisons:** Diffuse-only, specular-only, and combined renders, plus sRGB vs. linear-RGB output.

**References:**
- C. Colbert & J. Křivánek, *GPU-Based Importance Sampling*, GPU Gems 3, Ch. 20. [Link](https://developer.nvidia.com/gpugems/gpugems3/part-iii-rendering/chapter-20-gpu-based-importance-sampling)
- R. Ramamoorthi & P. Hanrahan, *An Efficient Representation for Irradiance Environment Maps*, SIGGRAPH 2001.

- 
![Image 6](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj3_0.png)
![Image 7](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj3_1.png)
![Image 8](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj3_2.png)
![Image 9](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj3_3.png)
![Image 10](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj3_4.png)
![Image 11](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj3_5.png)
![Image 12](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj3_6.png)
![Image 13](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj3_7.png)
![Image 14](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj3_8.png)

---

### Project 4 – Ambient Occlusion with Bilateral Filtering

This project adds **screen-space ambient occlusion** on top of the IBL lighting. It darkens crevices and contact areas that the skydome light shouldn't fully reach.

- **Inputs:** World position, normal, and depth come from the G-buffer.
- **AO estimator (Alchemy):** For each pixel, sample points within a radius of influence R are tested for occlusion. A depth threshold δ stops neighboring polygons from occluding each other, and a Heaviside function drops samples beyond R. The final factor is `A = max(0, 1 − s·S)^k`. The chosen settings are scale `s = 5` and contrast `k = 2`.
- **Sample pattern:** Samples follow a spiral around each pixel with a per-pixel hash rotation, `φ = (30x' ^ y') + 10x'y'`, which varies the sampling direction between pixels and prevents banding.
- **Edge-aware denoising:** A **bilateral filter**, run as separate horizontal and vertical passes, smooths the noise. It combines a Gaussian spatial kernel with a range kernel based on normal and depth similarity, so edges are preserved.
- **Application:** The AO factor scales only the ambient (IBL) light, not the direct lighting.
- **Debug views:** The raw AO, AO after the horizontal blur, AO after the vertical blur, and the final scene with and without AO.

**References:**
- M. McGuire et al., *The Alchemy Screen-Space Ambient Obscurance Algorithm*, HPG 2011. [Paper](https://faculty.digipen.edu/~gherron/references/References/AmbientOcclusion/VV11AlchemyAO.pdf)
- M. McGuire, M. Mara, D. Luebke, *Scalable Ambient Obscurance*, HPG 2012 (sample selection, eqs. 6–8). [Paper](https://faculty.digipen.edu/~gherron/references/References/AmbientOcclusion/McGuire12AO.pdf)

![Image 15](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj4_0.png)
![Image 16](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj4_1.png)
![Image 17](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj4_2.png)
![Image 18](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj4_3.png)
![Image 19](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj4_4.png)
![Image 20](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj4_5.png)
![Image 21](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj4_6.png)
![Image 22](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj4_7.png)
![Image 23](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj4_8.png)

---

### Project 5 – Summed-Area Table Shadow Filtering (VSM & MSM)

This project replaces Project 2's fixed-width convolution blur with a **Summed-Area Table (SAT)**. With an SAT, the size of the shadow kernel can vary, which better approximates soft penumbrae.

- The SAT is computed from the moment shadow map. Each texel holds the sum of all texels above and to its left. The SAT algorithm is applied twice for smoother shadows, and each pass is verified with RenderDoc captures.
- In the lighting fragment shader, any rectangular box filter is computed in **constant time** from four reads: `Sum = (D − B − C + A) / (4 · blurWidth²)`.
- The filtered moments feed both **VSM** (Chebyshev upper bound) and **MSM** (4-moment polynomial solve). MSM gives more accurate penumbra transitions with less light bleeding.
- The project also uses techniques to minimize shadow acne and light bleeding.

**References:**
- T. Scheuermann & J. Hensley, *Summed-Area Tables and Their Application to Dynamic Glossy Environment Reflections*, GDC 2005. [Slides](https://faculty.digipen.edu/~gherron/references/References/SAT/GDC2005_SATEnvironmentReflections.pdf)
- F. Crow, *Summed-Area Tables for Texture Mapping*, SIGGRAPH 1984.
- W. Donnelly & A. Lauritzen, *Variance Shadow Maps*, I3D 2006.
- C. Peters & R. Klein, *Moment Shadow Mapping*, I3D 2015.
  
![Image 24](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj5_0.png)
![Image 25](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj5_1.png)
![Image 26](https://github.com/durgasandeepn/ADVANCED-REAL-TIME-RENDERING-/blob/main/CS562-framework/images/Prj5_2.png)


---
