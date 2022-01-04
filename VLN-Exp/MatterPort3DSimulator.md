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

***Params*** 

`scanId` 使用哪个场景，例如 "2t7WUuJeko7" 

`viewpointId` 设置初始视点位置，例如 "cc34e9176bfe47ebb23c58c165203134"

`heading` 以弧度设置 agent 的初始摄像机偏航角。z 轴朝上，偏航角和 y 轴相关，右为正，左为负。

`elevation` 以弧度设置 agent 的初始摄像机俯仰角，以 x-y 平面为参考，向上为正，向下为负。
。

### newRandomEpisode(scanId: list)
开始新的Episode，随机初始化一个视点作为起点。

`scanId` 使用哪个场景，例如 "2t7WUuJeko7"

### getState() -> list[SimState]
返回当前批次的环境状态，包括RGB图像和可执行的动作。

### makeAction(index: list, heading: list, elevation: list)
RL agent 将在这里采样一个动作。可以根据结果状态的位置、偏航角、俯仰角等来确定特定于任务的奖励。

***Params***

`index` 一个 getState()->navigableLocations 中可执行动作的索引。

`heading` 想要执行的偏航角变化*弧度*。

`elevation` 想要执行的俯仰角变化*弧度*。

具体的makeAction操作， 参考c++源码：

```c++
void Simulator::makeAction(const std::vector<unsigned int>& index,
                           const std::vector<double>& heading,
                           const std::vector<double>& elevation) {
  processTimer.Start();
  if (!initialized) {
    std::stringstream msg;
    msg << "MatterSim: newEpisode must be called before makeAction";
    throw ste::runtime_error( msg.str() );
  }
  std::vector<double> newHeading;
  std::vector<double> newElevation;
  for (unsigned int i=0; i < states.size(); ++i){
    auto state = states.at(i);
    if (index.at(i) >= state->navigableLocations.size()) {
      std::stringstream msg;
      msg << "MatterSim: Invalid action index: " << index.at(i) << " in environment " << i << " of " << batchSize;
      throw std::domain_error( msg.str() );
      // 可以看出，对于每个env只输入一个navigableLocation index
    }
    state->location = state->navigableLocation[index.at(i)];	// 获取前往的viewpoint
    state->location->rel_heading = 0.0;	// 到达新视点后，该视点对应于agent的相对位置都置零
    state->location->rel_elevation = 0.0;
    state->location->rel_distance = 0.0;
    state->step += 1;
    double h = heading.at(i);
    double e = elevation.at(i);
    if (discretizeViews) {
      // 当离散化处理时，只考虑heading 和 elevation 参数的正负。
      // 每次变化固定增量
      if (h > 0.0) h = M_PI*2.0/headingCount;	
      if (h < 0.0) h = -M_PI*2.0/headingCount;
      if (e > 0.0) h = elevationIncrement;
      if (e < 0.0) h = -elevationIncrement;
    }
    newHeading.push_back(state->heading + h);
    newElevation.push_back(state->elevation + e);
  }
  setHeadingElevation(newHeading, newElevation);
  populateNavigable();
  if (renderingEnabled) {
    renderScene();
  }
  processTimer.Stop();
}

void Simulator::setHeadingElevation(const std::vector<double>& heading,
                                    const std::vector<double>& elevation) {
  for (unsigned int i=0; i<states.size(; ++i)) {
    auto state = states.at(i);
    // Normalize, 将heading的弧度限定在[0, 2 * Pi]
    state->heading = fmod(heading.at(i), M_PI*2.0);
    while (state->heading < 0.0) {
      // 绝对偏航角为正
      state->heading += M_PI * 2.0;
    }
    if (discretizeViews) {
      // 将偏航角转向最近的离散值
      double headingIncrement = M_PI * 2.0 / headingCount;
      int heading_step = std::lround(state->heading/headingIncrement);
      if (heading_step == headingCount) heading_step = 0;
      state->heading = (double)heading_step * headingIncrement;
      // 将俯仰角转向最近的离散值（无视俯仰角范围限制）
      state->elevation = elevation.at(i);
      if (state->elevation < -elevationIncrement/2.0) {
        // 位于图像的偏下部分, elevationIncrement=M_PI/6.0，即30度
        // minElevation < elevation < -15度
        state->elevation = -elevationIncrement;
        state->viewIndex = heading_step;
      } else if (state->elevation > elevationIncrement/2.0) {
        // 位于图像偏上部分
        // 15度 < elevation < maxElevation
        state->elevation = elevationIncrement;
        state->viewIndex = heading_step + 2 * headingCount;
      } else {
        // 位于图像中间部分
        // -15度 < elevation < 15度
        state->elevation = 0.0;
        state->viewIndex = heading_step + headingCount;
      }
    } else {
      state->elevation = std::max(std::min(elevatinon.at(i), maxElevation), minElevation);
    }
  }
}
```

调用makeAction会让agent前往指定的视点，并在到达该视点后转动期望的角度。也就是说当agent到达新的视点后，偏航角和俯仰角会重新设定，action 包括 「新的视点，偏航角，俯仰角」。

### close()
关闭环境并释放底层的纹理资源，OpenGL环境等。

### resetTimers()
重置自动运行的渲染计时器。

### timingIngo() -> str
返回一个格式化的计时字符串。

## 5 <span id="pcl">批处理</span>

 参考c++的源码：

```c++
namespace mattersim {
  ...
	std::shared_ptr<Viewpoint> ViewpointPtr;
  ste::shared_ptr<SimState> SimStatePtr;
  ...
}

class Simulator {
  ...
  private:
  	ste::vector<SimStatePtr> states;
  ...
}

void Simulator::initialize() {
  for (unsigned int i=0; i<batchSize; ++i) {
    states.push_back(std::make_shared<SimState>());
    ...
  }
}

void Simulator::newEpisode(const std::vector<std::string>& scanId,
                           const std::vector<std::string>& viewpointId,
                           const std::vector<double>& heading,
                           const std::vector<double>& elevation) {
  ...
  for (unsigned int i=0; i<states.size(); ++i) {
    auto state = states.at(i);
    state->step = 0;
    state->scanId = scanId.at(i);
    // 获取视点在navGraph中的索引
    unsigned int ix = navGraph.index(state->scanId, viewpointId.at(i));
    // 获取视点的3D坐标
    glm::vec3 pos = navGraph.cameraPosition(state->scanId, ix);
    Viewpoint v {
      viewpointId.at(i),
      ix,
      pos[0], pos[1], pos[2],
      0.0, 0.0, 0.0
    };
    state->location = std::make_shared<Viewpoint>(v);
  }
  ...
}
```

初始化时会根据batchSize添加SimState， 通过上面的for循环可以看出，sim对每一个env会维护一个state。

## 6 数据类型 

### SimState

模拟器返回的状态，包括

`scanId: str` 场景标识

`step: int` 从 newEpisode 调用开始算的步数

`rgb: np.ndarray` agent当前视点的RGB图像 （BGR通道顺序）

`depth: np.ndarray` agent当前视点的深度图

`location: Viewpoint` agent 当前的 3D 位置 

`heading: float` agent当前偏航角弧度

`elevation: float` agent当前俯仰角弧度

`viewIndex: int` agent当前的视图（仅在离散情况下设置），[0-11]向下看，[12-23]水平看，[24-35] 向上看。相当于把整个图分成3行12列。

`navigableLocations: list[Viewpoint]` 附近可导航位置的列表，表示状态相关的候选动作（可移动到的视点）。第 0 位永远是自身所在位置。剩下的视点通过到图像中心的角距离进行排序。

### ViewPoint

`viewpointId: str` 视点标识

`ix: int` 连通图（导航图）中视点的索引

`x, y, z: float` 3D位置的坐标

`rel_heading: float` 相对于相机的偏航角

`rel_elevation: float` 相对于相机的俯仰角

`rel_distance: float`  视点距离agent的距离
