# Working Log in Group_GS

## Webs Used

1. [nerfstudio official document](https://docs.nerf.studio/)

## Nerfstudio Start

### Installation

[Installation Instruction](https://docs.nerf.studio/quickstart/installation.html)

1. Create environment:

    ```text
    conda create --name nerfstudio -y python=3.8
    conda activate nerfstudio
    pyhton -m pip install --upgrade pip
    ```

2. Dependencies:
    - PyTorch
        - PyTorch
        - Build necessary CUDA extensions.

            ```text
            conda install -c "nvidia/label/cuda-11.8.0" cuda-toolkit
            ```

    - tiny-cuda-nn

        ```text
        pip install ninja git+https://github.com/NVlabs/tiny-cuda-nn/#subdirectory=bindings/torch
        ```

3. Installing nerfstudio

    ```text
    pip install nerfstudio
    ```

### Run First Model

[Training Your First Model](https://docs.nerf.studio/quickstart/first_nerf.html)

1. Train and run viewer:
    - Test dataset downloading: original order `ns-download-data nerfstudio --capture-name=poster` is not effective. I've uploaded the dataset to github. [Dataset](https://github.com/Everett492/Group_GS/tree/main/nerf_first_model/data/nerfstudio/poster)
    - Train model: `ns-train nerfacto --data data/nerfstudio/poster`
    ![Training](./image_note/First_Training.png)
2. Use pretrained model:
    - Load:

        ```text
        ns-train nerfacto --data data/nerfstudio/poster --load-dir {outputs/.../nerfstudio_models}
        ```

    - Visualize existing run:

        ```text
        ns-viewer --load-config {outputs/.../config.yml}
        ```

3. Export results
    - Rendering vedio:
        - Add camera --> Generate command --> input --> success
        - When testing in environment `nerfstudio`, problem occurred: `Could not find ffmpeg. Please install ffmpeg`. Using `pip install ffmpeg` didn't work. Use `sudo apt install ffmpeg` instead.
    - Point cloud: `ply` file
    - Mesh: `obj` file

## NeRF: Neural Radiance Field

### Theoretical Part

1. Original paper: [NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis](https://arxiv.org/pdf/2003.08934.pdf)
    - Inrtroduction:
        - A static scene is represented by a continuous `5D funtion` that outputs the `radiance` emitted in each direction (θ, φ) at each point (x, y, z) in space, and a `density` at each point which acts like a differential opacity controlling how much radiance is accumulated bt a ray passing through (x, y, z).
        - The 5D funtion is represented by an MLP.
        - Rendering step:
            - March camera rays through the scene to generate a sampled set of 3D points.
            - Use those points and their corresponding 2D viewing directions as input to the NN to produce an output set of colors and densities.
            - Use classical volume rendering techniques to accumulate those colors and densities into a 2D image.

            ![NeRF Rendering](./image_note/NeRF_Rendering.jpg)

        - Disadvantages of basic implementation:
            - low resolution, inefficient
            - solution 1: positional encoding
            - solution 2: hierarchical sampling
    - Neural Radiance Field Scence Representation
        - The 5D vector-valued function:
            - Input: (x, y, z, θ, φ)
            - Output: (r, g, b, σ)
            - Approximate this function with an MPL network: `FΘ : (x, d) → (c, σ)`.
        - Density σ is predicted as a function of only the location x. Color c is predicted as a function of both location and viewing direction.
        - Network Architecture:
        ![MLP Architecture](./image_note/NeRF_MLP.png)
    - Volume Rendering with Radiance Fields
        - The 5D neural rediance field repersents a scence as the volume density and directional emitted radiance at any point in sapce.
        - The volume density σ(x) can be interpreted as the differential probability of a ray termination at an infinitesimal particle at location x.
        - The expected color C(r) of camera ray r(t) = o + td with near and far bounds tn and tf is:
        ![Color C(r) Calculation](./image_note/Cr.png)
        - The function T(t) denotes accumulated transmittance along the ray from tn to t, the probability that the ray travels from tn to t without hitting any other particle.
        - Integral to Quadrature:
            - Stratified Sampling Approach: pertition [tn, tf] into N evenly-spaced bins and then dran one sample uniformly at random from within each bin
                ![Stratified Sampling](./image_note/Stratified_Sampling.png)
            - Use the samples above to estimate C(r):
                ![C(r) Estimating](./image_note/Cr_Estimate.png)
    - Optimizing a Neural Radiance Field
        - Positional encoding
            - The encoding function is applied separately to each of the three coordinate values in x and to the three components of the Cartesian viewing direction unit vector d.
            - For x, L = 10. For d, L = 4.
            ![Encoding Function](./image_note/NeRF_Encoding_Function.png)
        - Hierarchical volume sampling
        - Implementation details
        ![NeRF Loss Function](./image_note/NeRF_Loss_Function.png)
2. Camera parameters and coordinate system transformations
    - Reference: [Code Interpretation](https://zhuanlan.zhihu.com/p/593204605/)
    - Internal and external matrix of the camera:
        - External matrix:
            - Shape: a 4x4 matrix `M`, or world-to-camera matrix.
            - Useage: Transforms a point `P(world) = [x, y, z, 1]` in the world coordinate system to a point `P(camera) = MP(world)` in the camera coordinate system.
            - `M^(-1)`: c2w matrix:

            ```text
                       [r11, r12, r13, t1]
            [ R  T ] = [r21, r22, r23, t2]
            [ 0  1 ]   [r32, r32, r33, t3]
                       [  0,   0,   0,  1]
            ```

        - Internal matrix:
            - Usage: mapping 3D coordinates in the camera's coordinate system to the 2D image plane.
            - Take pinhhole camera as an example:

            ```text
                [fx,  0, cx]
            K = [ 0, fy, cy]
                [ 0,  0,  1]

            `fx` and `fy` are the horizontal and vertical focal lengths of the camera.
            `cx` and `cy` are the horizontal and vertical offsets of the image origin relative to the camera's center.
            ```

            ```python
            # Code constructing K in run_nerf.py
            if K is None:
                K = np.array([
                    [focal, 0, 0.5*W],
                    [0, focal, 0.5*H],
                    [0, 0, 1]
                ])
            ```

            - Abtaining camera matrix:
                - Synthesized data
                - Real data use COLMAP and [img2poses.py](https://github.com/Fyusion/LLFF/blob/master/imgs2poses.py) by NeRF's author.
                - `pose_bounds.npy`: `load_llff.py` will read the `poses_bounds.npy` file directly to get the camera parameters. `poses_bounds.npy` is an `Nx17` matrix, where N is the number of images. There are 17 parameters per image. The first 15 of these parameters can be rearranged into a 3x5 matrix form `[R T [H W f].transpose()]`. The last two parameters, `near` and `far` are used to represent to bounds of the scene.
        - Construcing 3D space ray:
            - A ray consists of a starting point and a direction, the starting point being the center of the camera. The location of pixel points in the imaging plane is also needed.
            - The NeRF code constructs the ray in the camera coordinate system, and then transforms the ray to the world coordinate system using the camera-to-world (c2w) matrix.
            ![3D_Ray_Constructing_OpenCV](./image_note/3D_Ray_Constructing.png)
            - Code in NeRF:

                ```python
                def get_rays_np(H, W, K, c2w):
                    i, j = np.meshgrid(np.arange(W, dtype=np.float32), np.arange(H, dtype=np.float32), indexing='xy')
                    dirs = np.stack([(i-K[0][2])/K[0][0], -(j-K[1][2])/K[1][1], -np.ones_like(i)], -1)
                    # Rotate ray directions from camera frame to the world frame
                    rays_d = np.sum(dirs[..., np.newaxis, :] * c2w[:3,:3], -1)  # dot product, equals to: [c2w.dot(dir) for dir in dirs]
                    # Translate camera frame's origin to the world frame. It is the origin of all rays.
                    rays_o = np.broadcast_to(c2w[:3,-1], np.shape(rays_d))
                    return rays_o, rays_d
                ```

3. Code of the basic model:

    - Code: [basic_model.py](https://github.com/nerfstudio-project/nerfstudio/blob/main/nerfstudio/models/base_model.py)
    - Model defination:

### NeRFactor
