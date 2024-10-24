# frigate-proxmox-docker-openvino

Complete setup for OpenVINO hardware acceleration in Frigate, instead of using Coral.

This tutorial is adapted for the Docker version of Frigate installed in a Proxmox LXC and primarily focuses on GPU passthrough.

## Prerequisites

- Intel iX > GEN6 architecture (i.e., compatible with OpenVINO acceleration)
- A working Proxmox installation

Check in your PVE Shell that `/dev/dri/renderD128` is available:
> ```bash
> cd /dev/dri
> ls
> ```

Optionally, install Intel GPU tools:
> ```bash
> apt install intel-gpu-tools
> ```

Now you can check GPU access/usage:
> ```bash
> intel_gpu_top
> ```

It should lead to something like this:

![image](https://github.com/user-attachments/assets/0474d76c-e4c7-45df-8023-5dc10809c01c)

## Create Docker LXC

The easiest way is to use [Tteck's scripts](https://tteck.github.io/Proxmox/).

First, in the PVE console, launch Tteck's script to install a new Docker LXC:
> ```bash
> bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/ct/docker.sh)"
> ```

During installation:
- Switch to "advanced mode"
- Select Debian 12
- Make the LXC **PRIVILEGED**
- It's recommended to choose 8GB of RAM and 2 or 4 cores
- Add Portainer if needed
- Add Docker Compose

Once the LXC is created, you can also install intel-gpu-tools **inside** the LXC:
> ```bash
> apt install intel-gpu-tools
> ```

Next, you have to add GPU passthrough to the LXC to allow Frigate access to OpenVINO acceleration. On your LXC "Resources," add "Device Passthrough":

![image](https://github.com/user-attachments/assets/071007bb-ad90-43c9-92ac-0c79313b83eb)

Specify the path you want to add: 
> `/dev/dri/renderD128`.

**Reboot** the LXC, and it should now have access to the GPU.


---


## Test iGPU Passthrough
### Confirm Access in the LXC

To ensure your privileged LXC with GPU passthrough is properly configured, follow these steps:

1. **Verify iGPU Device in the LXC:**
   Run the following command to check if the iGPU device is visible inside the LXC container:
   > ```bash
   > ls /dev/dri
   > ```

  > [!NOTE]
  > Expected output should include `card0` and `renderD128`.

2. **Install Intel GPU Tools:**
   If `intel-gpu-tools` isn't installed, run the following:
   > ```bash
   > apt update
   > apt install intel-gpu-tools
   > ```

3. **Monitor iGPU with `intel_gpu_top`:**
   Test if the iGPU is active and accessible by running:
   > ```bash
   > intel_gpu_top
   > ```

  > [!NOTE]
  > _This should display statistics like GPU load, memory reads/writes, etc._

### Confirm Access in Docker

1. **Pull the OpenVINO Runtime Docker Image:**
   > ```bash
   > docker pull openvino/ubuntu20_runtime:latest
   > ```

2. **Run Docker Container with iGPU Passthrough:**
   > ```bash
   > docker run --device /dev/dri:/dev/dri -it openvino/ubuntu20_runtime:latest
   > ```

3. **Run a Sample Inference Test inside the Docker Container:**
   > ```bash
   > source /opt/intel/openvino_2022/setupvars.sh
   > python3 /opt/intel/openvino_2022/deployment_tools/open_model_zoo/demos/classification_demo/python/classification_demo.py -m /opt/intel/openvino_2022/deployment_tools/open_model_zoo/models/public/squeezenet1.1/squeezenet1.1.xml -i /opt/intel/openvino_2022/deployment_tools/demo/car.png
   > ```

  > [!NOTE]
  > _This command runs a basic image classification using the iGPU, confirming your setup._


---


## Frigate Docker

### Create Folders

On the LXC shell, create folders to organize your Frigate storage for videos, captures, models, and configs.

Here are my usual settings:
> ```bash
> mkdir /opt/frigate
> mkdir /opt/frigate/media
> mkdir /opt/frigate/config
> ```

Create the folders according to your needs.

Next, we will build the Docker container.

Create a `docker-compose.yml` in the root folder:
> ```bash
> cd /opt/frigate
> nano docker-compose.yml
> ```

Alternatively, create a stack in Portainer:

![image](https://github.com/user-attachments/assets/3a4add3d-38ec-4313-a9b1-d6761057726c)

Add the following content:
> ```yaml
> version: "3.9"
> services:
>   frigate:
>     container_name: frigate
>     privileged: true
>     restart: unless-stopped
>     image: ghcr.io/blakeblackshear/frigate:0.14.1
>     cap_add:
>       - CAP_PERFMON
>     shm_size: "256mb"
>     devices:
>       - /dev/dri/renderD128:/dev/dri/renderD128
>       - /dev/dri/card0:/dev/dri/card0
>     volumes:
>       - /etc/localtime:/etc/localtime:ro
>       - /opt/frigate/config:/config
>       - /opt/frigate/media:/media/frigate
>       - type: tmpfs
>         target: /tmp/cache
>         tmpfs:
>           size: 1G
>     ports:
>       - "5000:5000"
>       - "8971:8971"
>       - "1984:1984"
>       - "8554:8554" # RTSP feeds
>       - "8555:8555/tcp" # WebRTC over TCP
>       - "8555:8555/udp" # WebRTC over UDP
>     environment:
>       FRIGATE_RTSP_PASSWORD: ****
>       PLUS_API_KEY: ****
> ```

As you can see:
- The container is **privileged**
- `/dev/dri/renderD128` is passed through from the LXC to the container
- Created folders are bound to Frigate's usual folders
- `shm_size` has to be set according to [the documentation](https://docs.frigate.video/frigate/installation/#calculating-required-shm-size)
- `tmpfs` has to be adjusted to your configuration; see [the documentation](https://docs.frigate.video/frigate/installation/#storage)
- Ports for `UI`, `RTSP`, and `WebRTC` are forwarded
- Define the `FRIGATE_RTSP_PASSWORD` and `PLUS_API_KEY` if needed

At this point, the Docker container is ready and has access to the GPU.

> [!CAUTION]
> 🚫 **DO NOT START IT YET** because *you still need to provide the Frigate configuration!* 🚫


---


## Setup Frigate for OpenVINO Acceleration

Add your Frigate configuration:
> ```bash
> cd /opt/frigate/config
> nano config.yml
> ```

Edit it according to your setup. Now, you must add the [following lines](https://docs.frigate.video/configuration/object_detectors/#openvino-detector) to your Frigate config:
> ```yaml
> detectors:
>   ov:
>     type: openvino
>     device: GPU
> 
> model:
>   width: 300
>   height: 300
>   input_tensor: nhwc
>   input_pixel_format: bgr
>   path: /openvino-model/ssdlite_mobilenet_v2.xml
>   labelmap_path: /openvino-model/coco_91cl_bkgr.txt
> ```

Once your `config.yml` is ready, build your container by running `docker compose up` or "Deploy Stack" if you're using Portainer.

Reboot everything, and go to the Frigate UI to check that everything is working:

![image](https://github.com/user-attachments/assets/abad95f0-f0c9-4b59-853f-56a252e6bb65)

> !NOTE]
> You should observe:
> - Low inference time (~20 ms)
> - Low CPU usage
> - GPU usage

You can also check with `intel_gpu_top` inside the LXC console and see that Render/3D has some load according to Frigate detections:

![image](https://github.com/user-attachments/assets/ce307fb5-e856-4846-b1ee-94a6bda9758a)

On your Proxmox dashboard, you can see that the CPU load of the LXC is drastically reduced:

![image](https://github.com/user-attachments/assets/365406eb-f0bc-4367-ba31-42609c587d87)


---


## Extra Settings

### CPU Load

> [!TIP]
> I found experimentally that running these two Tteck scripts in the PVE console greatly reduces CPU consumption in "idle mode" (i.e., when Frigate is only observing and no detections are running):
> - [Filesystem Trim](https://tteck.github.io/Proxmox/#proxmox-ve-lxc-filesystem-trim)
> - [CPU Scaling Governor](https://tteck.github.io/Proxmox/#proxmox-ve-cpu-scaling-governor): _Set the governor to_ `powersave`.
> 
> Experiment on your own!

### YOLO NAS Models

In addition to the default SSDLite model, [YOLO NAS](https://github.com/Deci-AI/super-gradients) models are also [available for OpenVINO acceleration](https://docs.frigate.video/configuration/object_detectors/#yolo-nas).

To use it, you must build the model to make it compatible with Frigate. This can easily be done with the dedicated [Google Colab](https://colab.research.google.com/github/blakeblackshear/frigate/blob/dev/notebooks/YOLO_NAS_Pretrained_Export.ipynb).

The only thing to do is define the dimensions of the input image shape. While 320x320 leads to higher inference time, I recommend using 256x256.
> ```
> input_image_shape=(256,256),
> ```

Select the base precision of the model. The **S** version is good enough; the **M** version induces much higher inference time:
> ```
> model = models.get(Models.YOLO_NAS_S, pretrained_weights="coco")
> ```

> [!TIP]
> *You can experiment to find the right combination for your hardware. Try to keep the inference time around 20 ms.*

Specify the name of the model file you will generate:
> ```
> files.download('yolo_nas_s.onnx')
> ```

Now, simply launch all the steps in Colab, one by one, and it will download the model file:

![image](https://github.com/user-attachments/assets/53a211a9-c2e9-4a9d-bff2-946ce674fe27)

Copy the model file you generated to your Frigate config folder `/opt/frigate/config`.

Then, modify your detector settings accordingly:
> ```yaml
> detectors:
>   ov:
>     type: openvino
>     device: GPU
> 
> model:
>   model_type: yolonas
>   width: 256 # <--- should match whatever was set in the notebook
>   height: 256 # <--- should match whatever was set in the notebook
>   input_tensor: nchw # <--- note that this differs from the setting for the SSDLite model!
>   input_pixel_format: bgr
>   path: /config/yolo_nas_s_256.onnx # <--- should match the path and name of your model file
>   labelmap_path: /labelmap/coco-80.txt # <--- should match the name and location of the COCO80 labelmap file
> ```

> [!NOTE]
> YOLO NAS uses the COCO80 labelmap instead of COCO91.*

Restart, and voilà!
