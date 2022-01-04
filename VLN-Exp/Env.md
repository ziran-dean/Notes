# Batched Room-to-Room navigation environment
（以VLN-BERT代码为例）

[toc]

## Class: EnvBatch

### \__init__

```python
class EnvBatch():
    ''' A simple wrapper for a batch of MatterSim environments,
        using discretized viewpoints and pretrained features '''

    def __init__(self, feature_store=None, batch_size=100):
        """
        1. Load pretrained image feature
        2. Init the Simulator.
        :param feature_store: The name of file stored the feature.
        :param batch_size:  Used to create the simulator list.
        """
        if feature_store:
          	# 预加载 image feature -- 通过预训练的模型编码得到
            if type(feature_store) is dict:     # A silly way to avoid multiple reading
                self.features = feature_store
                self.image_w = 640	# 分辨率 640 x 480
                self.image_h = 480
                self.vfov = 60
                self.feature_size = next(iter(self.features.values())).shape[-1]
                print('The feature size is %d' % self.feature_size)
        else:
            print('    Image features not provided - in testing mode')
            self.features = None
            self.image_w = 640
            self.image_h = 480
            self.vfov = 60
        self.sims = []
        for i in range(batch_size):	
            sim = MatterSim.Simulator()
            sim.setRenderingEnabled(False)	# 禁用渲染
            sim.setDiscretizedViewingAngles(True)   # Set increment/decrement to 30 degree. (otherwise by radians) 
            # 离散化的视角变化，变化限定为 30 度
            sim.setCameraResolution(self.image_w, self.image_h)	# 设置相机分辨率
            sim.setCameraVFOV(math.radians(self.vfov))	# 设置相机的垂直视角
            sim.initialize()	# 初始化模拟器
            self.sims.append(sim)	# 加入到 env batch 中
```

**EnvBatch** 通过 list 的方式来存储一个批次的 Env。可以预先读取处理好的视觉特征。

### newEpisodes

```python
def newEpisodes(self, scanIds, viewpointIds, headings):
        for i, (scanId, viewpointId, heading) in enumerate(zip(scanIds, viewpointIds, headings)):
            self.sims[i].newEpisode([scanId], [viewpointId], [heading], [0])
```

初始化batch中env的起点。

