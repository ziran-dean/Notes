# MP3D Sim API
[toc]

## 1 基础配置 
### setDatasetPath(path: str)
设置MatterPort3D的数据集路径,该路径下包含"\<scanId\>/matterport_skybox_images/"。默认值为 "./data/v1/scans/"。


### setNavGraphPath(path: str)
设置viewpoint的联通图路径，路径下包含"/\<scanId\>_connectivity.json"。默认值为 "./connectivity"。

### serRenderingEnabled(value: bool)
启用或禁用渲染。用于测试。默认值为 True。

## 2 相机配置
### setCameraResolution(width: int, height: int)
设置相机分辨率。默认值为 320 x 320。

### setCameraVFOV(vfov: float)
设置相机垂直视角（以弧度为单位）。默认值为0.8，大约46度。

![20211230105044-2021-12-30-10-50-44](https://raw.githubusercontent.com/ziran-dean/picbed/main/images/20211230105044-2021-12-30-10-50-44.png)

### setElevationLimits(min: float, max: float)
设置相机仰角的最小和最大限制弧度。默认值是+-0.94弧度。

### setDiscretizedViewingAngles(valur: bool)
启用或禁用离散化的视角。当启用时，偏航角(heading)和俯仰角(elevation)的变化将被限制为从零开始30度的增量，左/右/上/下移动由 makeAction 偏航角和俯仰角参数的符号触发。默认值为 False。

## 3 实验配置
### setRestrictedNaviagtion(value: bool)
启用或者禁用受限的导航。当启用时 agent 只能导航到当前朝向相机视场中包含的附近视点，禁用时 agent 可以导航到导航图中任意邻接的试点。默认值为 True。

### setPreloadingEnabled(value: bool)
启用或禁用将图片从磁盘预加载到内存中。默认值为 False。对于训练模型来说，启用更好，但会使模拟器启动时间变长。

### setDepthEnabled(value: bool)
启用或禁用深度图像的渲染。默认为false(禁用)。

### setBatchSize(size: int)
设置在一个批次中环境的数量。默认值是1。

### setCacheSize(size: int)
设置GPU内存中用于储存全景图的缓存大小。默认值是200。应该要比 batch size 大很多。

### setSeed(seed: int)
为还没有提供视点的 episodes 设置随机种子。

## 4 实验操作
### initialize()
初始化模拟器。之后进行的配置将不起作用。

### newEpisode(scanId: list, viewpointId: list, heading: list, elevation: list)
开始新的Episode。如果没有提供视点，随机初始化一个视点作为起点。
这里的 list 对应一个 batch 中不同的 Envs。
***Params*** 
`scanId` 使用哪个场景，例如 "2t7WUuJeko7"
`viewpointId` 设置初始视点位置，例如 "cc34e9176bfe47ebb23c58c165203134"
`heading` 以弧度设置 agent 的初始摄像机偏航角。z 轴朝上，偏航角和 y 轴相关，右为正，左为负。
`elevation` 以弧度设置 agent 的初始摄像机俯仰角，以 x-y 平面为参考，向上为正，向下为负。
。

### newRandomEpisode(scanId: list)
开始新的Episode，随机初始化一个视点作为起点。
`scanId` 使用哪个场景，例如 "2t7WUuJeko7"

### getState() -> list
返回当前批次的环境状态，包括RGB图像和可执行的动作。

### makeAction(index: list, heading: list, elevation: list)
RL agent 将在这里对动作进行采样。可以根据结果状态的位置、偏航角、俯仰角等来确定特定于任务的奖励。
***Params***
`index` 可执行动作的索引，由 getState()->navigableLocations 给出。
`heading` 想要执行的偏航角变化*弧度*。
`elevation` 想要执行的俯仰角变化*弧度*。

### close()
关闭环境并释放底层的纹理资源，OpenGL环境等。

### resetTimers()
重置自动运行的渲染计时器。

### timingIngo() -> str
返回一个格式化的计时字符串。
