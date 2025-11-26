# Patch-Based Inpainting Algorithm (Abstract Pseudocode)

This document provides abstract pseudocode for an exemplar-based inpainting algorithm,
similar to techniques used in professional image editing software.

**Reference:** Criminisi, A., Pérez, P., & Toyama, K. (2004).
"Region filling and object removal by exemplar-based image inpainting."

---

## Core Data Structures

```
Structure ImageRegion:
    pixels: 2D array of color values
    width, height: dimensions

Structure Mask:
    values: 2D array of {0, 1} or {true, false}
    // 1/true = region to fill (target)
    // 0/false = source region (known pixels)

Structure Patch:
    centerX, centerY: coordinates
    size: integer (e.g., 9 for 9×9 patch)
    pixels: 2D array extracted from image

Structure Priority:
    confidence: float [0, 1]
    data: float [0, 1]
    total: float = confidence × data
```

---

## Main Algorithm

```pseudocode
ALGORITHM PatchBasedInpainting(image, mask, patchSize, searchRadius)

    INPUT:
        image: original image with region to inpaint
        mask: binary mask (1 = fill, 0 = known)
        patchSize: size of patch (e.g., 9 means 9×9)
        searchRadius: how far to search for matching patches

    OUTPUT:
        inpainted_image: result with filled region

    // Initialize confidence map
    confidence_map ← InitializeConfidenceMap(mask)
    // Pixels outside mask have confidence = 1.0
    // Pixels inside mask have confidence = 0.0

    fill_front ← ExtractBoundary(mask)
    // Boundary = border pixels between filled and unfilled regions

    WHILE fill_front is not empty DO

        // Step 1: Compute priorities for all boundary patches
        priorities ← ComputePriorities(
            image,
            mask,
            confidence_map,
            fill_front,
            patchSize
        )

        // Step 2: Select patch with highest priority
        target_patch ← SelectHighestPriorityPatch(priorities, fill_front)

        // Step 3: Find best matching source patch
        source_patch ← FindBestMatch(
            image,
            mask,
            target_patch,
            patchSize,
            searchRadius
        )

        // Step 4: Copy pixels from source to target
        CopyPatchData(
            image,
            mask,
            confidence_map,
            source_patch,
            target_patch
        )

        // Step 5: Update boundary
        fill_front ← UpdateBoundary(mask)

    END WHILE

    // Optional: Apply Poisson blending for smoothing
    result ← PoissonBlend(image, mask)

    RETURN result

END ALGORITHM
```

---

## Priority Computation

```pseudocode
FUNCTION ComputePriorities(image, mask, confidence_map, boundary, patchSize)

    priorities ← empty list

    FOR EACH pixel p IN boundary DO

        // Extract patch centered at p
        patch ← ExtractPatch(image, p, patchSize)

        // Compute confidence term
        C(p) ← ComputeConfidenceTerm(patch, mask, confidence_map)

        // Compute data term (edge strength)
        D(p) ← ComputeDataTerm(patch, mask, image)

        // Total priority
        P(p) ← C(p) × D(p)

        priorities.add(p, P(p))

    END FOR

    RETURN priorities

END FUNCTION
```

---

## Confidence Term

```pseudocode
FUNCTION ComputeConfidenceTerm(patch, mask, confidence_map)

    // Confidence = average confidence of known pixels in patch

    sum_confidence ← 0
    count ← 0

    FOR EACH pixel (x, y) IN patch DO
        IF mask[x, y] == KNOWN THEN
            sum_confidence ← sum_confidence + confidence_map[x, y]
            count ← count + 1
        END IF
    END FOR

    IF count > 0 THEN
        confidence ← sum_confidence / (patchSize × patchSize)
    ELSE
        confidence ← 0
    END IF

    RETURN confidence

END FUNCTION
```

---

## Data Term (Isophote Strength)

```pseudocode
FUNCTION ComputeDataTerm(patch, mask, image)

    // Data term measures strength of edges/structures
    // perpendicular to the fill front

    center ← patch.center

    // Compute image gradient at boundary
    gradient ← ComputeGradient(image, center)
    grad_x ← gradient.x
    grad_y ← gradient.y

    // Compute normal to boundary (fill front)
    normal ← ComputeBoundaryNormal(mask, center)
    norm_x ← normal.x
    norm_y ← normal.y

    // Isophote direction (perpendicular to gradient)
    isophote_x ← -grad_y
    isophote_y ← grad_x

    // Data term = dot product of isophote and normal
    data_term ← |isophote_x × norm_x + isophote_y × norm_y|

    // Normalize by maximum gradient magnitude
    max_gradient ← ComputeMaxGradient(image)
    data_term ← data_term / max_gradient

    RETURN data_term

END FUNCTION
```

---

## Gradient Computation

```pseudocode
FUNCTION ComputeGradient(image, point)

    // Use Sobel operators or simple finite differences

    x ← point.x
    y ← point.y

    // Sobel kernels
    sobel_x ← [[-1, 0, 1],
               [-2, 0, 2],
               [-1, 0, 1]]

    sobel_y ← [[-1, -2, -1],
               [ 0,  0,  0],
               [ 1,  2,  1]]

    grad_x ← Convolve(image, sobel_x, x, y)
    grad_y ← Convolve(image, sobel_y, x, y)

    RETURN (grad_x, grad_y)

END FUNCTION
```

---

## Boundary Normal Computation

```pseudocode
FUNCTION ComputeBoundaryNormal(mask, point)

    // Normal points from known region into unknown region

    x ← point.x
    y ← point.y

    // Simple gradient of mask
    norm_x ← (mask[x+1, y] - mask[x-1, y]) / 2
    norm_y ← (mask[x, y+1] - mask[x, y-1]) / 2

    // Normalize to unit vector
    length ← sqrt(norm_x² + norm_y²)

    IF length > 0 THEN
        norm_x ← norm_x / length
        norm_y ← norm_y / length
    END IF

    RETURN (norm_x, norm_y)

END FUNCTION
```

---

## Patch Matching (Best Exemplar Search)

```pseudocode
FUNCTION FindBestMatch(image, mask, target_patch, patchSize, searchRadius)

    best_match ← NULL
    min_distance ← INFINITY

    target_center ← target_patch.center

    // Define search window around target
    search_window ← DefineSearchWindow(
        target_center,
        searchRadius,
        image.dimensions
    )

    FOR EACH candidate_center IN search_window DO

        // Skip if candidate overlaps with unknown region
        candidate_patch ← ExtractPatch(image, candidate_center, patchSize)

        IF PatchOverlapsUnknown(candidate_patch, mask) THEN
            CONTINUE
        END IF

        // Compute patch similarity (SSD or SAD)
        distance ← ComputePatchDistance(
            target_patch,
            candidate_patch,
            mask
        )

        IF distance < min_distance THEN
            min_distance ← distance
            best_match ← candidate_patch
        END IF

    END FOR

    RETURN best_match

END FUNCTION
```

---

## Patch Distance (SSD with Valid Pixels Only)

```pseudocode
FUNCTION ComputePatchDistance(patch_a, patch_b, mask)

    // Sum of Squared Differences (SSD)
    // Only compare pixels that are known in target patch

    sum ← 0
    count ← 0

    half_size ← patchSize / 2

    FOR dy ← -half_size TO half_size DO
        FOR dx ← -half_size TO half_size DO

            x_a ← patch_a.center.x + dx
            y_a ← patch_a.center.y + dy

            x_b ← patch_b.center.x + dx
            y_b ← patch_b.center.y + dy

            // Only compare if pixel in target is known
            IF mask[x_a, y_a] == KNOWN THEN

                pixel_a ← patch_a.pixels[dx, dy]
                pixel_b ← patch_b.pixels[dx, dy]

                // Color distance (can be RGB, Lab, etc.)
                diff ← ColorDistance(pixel_a, pixel_b)
                sum ← sum + diff²
                count ← count + 1

            END IF

        END FOR
    END FOR

    IF count > 0 THEN
        distance ← sum / count  // Normalized SSD
    ELSE
        distance ← INFINITY
    END IF

    RETURN distance

END FUNCTION
```

---

## Color Distance

```pseudocode
FUNCTION ColorDistance(color_a, color_b)

    // Euclidean distance in RGB space
    // (Can use Lab or other color spaces for better perceptual accuracy)

    dr ← color_a.red - color_b.red
    dg ← color_a.green - color_b.green
    db ← color_a.blue - color_b.blue

    distance ← sqrt(dr² + dg² + db²)

    RETURN distance

END FUNCTION
```

---

## Copy Patch Data

```pseudocode
FUNCTION CopyPatchData(image, mask, confidence_map, source, target)

    // Copy pixels from source patch to target patch
    // Only copy pixels that are currently unknown in target

    half_size ← patchSize / 2

    // Get confidence of target patch (for updating confidence map)
    target_confidence ← ComputeConfidenceTerm(target, mask, confidence_map)

    FOR dy ← -half_size TO half_size DO
        FOR dx ← -half_size TO half_size DO

            x_src ← source.center.x + dx
            y_src ← source.center.y + dy

            x_tgt ← target.center.x + dx
            y_tgt ← target.center.y + dy

            // Only fill pixels that are currently unknown
            IF mask[x_tgt, y_tgt] == UNKNOWN THEN

                // Copy color
                image[x_tgt, y_tgt] ← image[x_src, y_src]

                // Update mask
                mask[x_tgt, y_tgt] ← KNOWN

                // Update confidence
                confidence_map[x_tgt, y_tgt] ← target_confidence

            END IF

        END FOR
    END FOR

END FUNCTION
```

---

## Boundary Extraction and Update

```pseudocode
FUNCTION ExtractBoundary(mask)

    // Boundary = pixels in unknown region adjacent to known region

    boundary ← empty set

    FOR y ← 0 TO height - 1 DO
        FOR x ← 0 TO width - 1 DO

            IF mask[x, y] == UNKNOWN THEN

                // Check 4-neighbors or 8-neighbors
                has_known_neighbor ← FALSE

                FOR EACH neighbor (nx, ny) IN Neighbors(x, y) DO
                    IF mask[nx, ny] == KNOWN THEN
                        has_known_neighbor ← TRUE
                        BREAK
                    END IF
                END FOR

                IF has_known_neighbor THEN
                    boundary.add((x, y))
                END IF

            END IF

        END FOR
    END FOR

    RETURN boundary

END FUNCTION
```

---

## Poisson Blending (Optional Smoothing)

```pseudocode
FUNCTION PoissonBlend(image, original_mask)

    // Seamlessly blend filled region using Poisson equation
    // Minimize: Δf = Δg (Laplacian matching)
    // Subject to boundary conditions

    // This is typically solved using:
    // - Direct methods (solve linear system Ax = b)
    // - Iterative methods (Jacobi, Gauss-Seidel, multigrid)

    result ← copy of image

    // Build Laplacian system
    FOR EACH pixel (x, y) IN original_mask DO

        // Compute guidance field (gradient from filled image)
        grad_x ← image[x+1, y] - image[x-1, y]
        grad_y ← image[x, y+1] - image[x, y-1]

        // Solve for pixel value that matches gradients
        // while respecting boundary conditions

    END FOR

    // Solve system iteratively
    max_iterations ← 1000

    FOR iteration ← 1 TO max_iterations DO

        convergence ← 0

        FOR EACH pixel (x, y) IN original_mask DO

            // Average of 4-neighbors minus divergence
            old_value ← result[x, y]

            new_value ← (
                result[x-1, y] + result[x+1, y] +
                result[x, y-1] + result[x, y+1]
            ) / 4

            result[x, y] ← new_value

            convergence ← max(convergence, |new_value - old_value|)

        END FOR

        IF convergence < threshold THEN
            BREAK
        END IF

    END FOR

    RETURN result

END FUNCTION
```

---

## Algorithm Parameters and Tuning

```
TYPICAL PARAMETERS:

    patchSize: 9 to 15 pixels
        - Smaller: faster, more detail preservation
        - Larger: better structure continuation, slower

    searchRadius: 50 to 200 pixels (or entire image)
        - Larger: better matches, slower
        - Adaptive: start small, expand if no good match

    confidenceTerm weight: 1.0
        - Determines fill order priority

    dataTerm weight: α (typically 255 for 8-bit images)
        - Balance between confidence and structure

    Poisson iterations: 500 to 2000
        - More iterations = smoother blend
```

---

## Optimization Strategies

```pseudocode
OPTIMIZATIONS:

1. Hierarchical/Multiscale:
   - Coarse-to-fine pyramid
   - Fill at low resolution first
   - Refine at higher resolutions

2. Approximate Nearest Neighbor:
   - Use PatchMatch algorithm
   - Randomized search with propagation
   - Much faster than exhaustive search

3. GPU Acceleration:
   - Parallelize patch distance computation
   - Use texture sampling for patch extraction
   - Compute shaders for Poisson solver

4. Adaptive Patch Size:
   - Larger patches near straight edges
   - Smaller patches in textured regions

5. Caching:
   - Precompute gradients
   - Cache patch distances
   - Reuse search results

6. Early Termination:
   - Stop search if distance < threshold
   - Good enough match vs. perfect match
```

---

## Complexity Analysis

```
TIME COMPLEXITY:

    Let:
        n = number of pixels to fill
        k = patch size (k×k)
        s = search window size (s×s)

    Per iteration: O(s × k²)
        - Searching s candidate patches
        - Comparing k² pixels per patch

    Total: O(n × s × k²)

    With optimizations (PatchMatch): O(n × k² × log(s))

SPACE COMPLEXITY:

    O(image_size) for:
        - Original image
        - Confidence map
        - Mask
        - Intermediate results
```

---

## Advanced Concepts

```pseudocode
CONCEPT: Bidirectional Similarity

    // Not only find best source for target
    // But ensure target is also best match for source
    // Reduces repetitive patterns

    FUNCTION BidirectionalMatch(target, candidate):
        forward_distance ← Distance(target, candidate)
        backward_distance ← Distance(candidate, target)
        RETURN (forward_distance + backward_distance) / 2


CONCEPT: Structure Tensor

    // Better edge/structure detection than simple gradient

    FUNCTION ComputeStructureTensor(image, point):
        Ix ← ∂image/∂x
        Iy ← ∂image/∂y

        J11 ← Gaussian_blur(Ix × Ix)
        J12 ← Gaussian_blur(Ix × Iy)
        J22 ← Gaussian_blur(Iy × Iy)

        // Eigenvalues give edge strength and direction
        RETURN StructureTensor(J11, J12, J22)


CONCEPT: Coherence-Based Propagation

    // Fill patches in order that respects structure
    // Continue edges and textures smoothly

    FUNCTION PropagateCoherent(previous_source, current_target):
        // Prefer sources near previous source
        // Encourages coherent texture synthesis


CONCEPT: Color Transfer

    // Match color statistics between regions

    FUNCTION MatchColorStatistics(source, target):
        mean_s, std_s ← Statistics(source)
        mean_t, std_t ← Statistics(target)

        // Transfer statistics
        result ← (source - mean_s) × (std_t / std_s) + mean_t
        RETURN result
```

---

## References and Further Reading

```
PAPERS:

1. Criminisi et al. (2004)
   "Region Filling and Object Removal by Exemplar-Based Image Inpainting"
   IEEE TPAMI

2. Barnes et al. (2009)
   "PatchMatch: A Randomized Correspondence Algorithm"
   ACM SIGGRAPH

3. Pérez et al. (2003)
   "Poisson Image Editing"
   ACM SIGGRAPH

4. Bertalmio et al. (2000)
   "Image Inpainting"
   ACM SIGGRAPH

5. Efros & Leung (1999)
   "Texture Synthesis by Non-Parametric Sampling"
   ICCV


APPLICATIONS:

- Object removal in photos
- Restoration of damaged images
- Video inpainting (frame-by-frame)
- Virtual reality scene completion
- Medical image reconstruction
```

---

## Educational Notes

This pseudocode represents a **theoretical algorithm** for academic understanding.

**Key Insights:**
- Priority-driven filling preserves structure
- Patch-based approach captures texture
- Confidence propagation ensures quality
- Boundary-first strategy is crucial
- Balance between speed and quality

**Limitations:**
- Computationally intensive
- Requires good source material nearby
- May fail on complex structures
- Not suitable for large missing regions
- Struggles with geometric patterns

**Modern Approaches:**
- Deep learning methods (GANs, diffusion models)
- Learned patch representations
- Context encoders
- Semantic inpainting
