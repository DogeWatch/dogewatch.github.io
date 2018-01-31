---
layout:		post
title:		"APOLLO源码解读"
subtitle:	"PERCEPTION"
date:		2018-01-31
author:	"dogewatch"
header-img:	"img/post-bg-kuaidi.jpg"
tags:
    - Autopilot

---
# 前言

Perception（感知）模块是百度APOLLO自动驾驶框架的基础模块，它的主要功能是根据激光雷达Lidar，电磁波雷达Radar、摄像机Camera等传感器采集到的数据来感知周围环境，其主要任务是对障碍物（行人、车辆等）进行识别，预测障碍物轨迹、对交通信号灯进行识别等。官方对perception模块的整体介绍在[https://github.com/ApolloAuto/apollo/blob/master/modules/perception/README.md](https://github.com/ApolloAuto/apollo/blob/master/modules/perception/README.md)。

## 文件结构

模块结构如下图：

![img](/img/post/perception-1.png)

整个模块基于DAG（有向无环图）来控制并有序地执行任务，在DAG配置中有三个组件：子节点、节点与节点之间的连线以及共享数据。模块中的每个功能都是作为DAG中的一个子节点来实现的，这么做的好处是使得各个功能的实现更加模块化。

## perception/common

该目录下仅有两个perception_gflags.*文件，其中声明了很多变量并初始化了值，这些变量在其它文件中会被用到

perception/common/perception_gflags.h

```c++
DECLARE_string(perception_adapter_config_filename);

/// lib/config_manager/config_manager.cc
DECLARE_string(config_manager_path);
DECLARE_string(work_root);

/// obstacle/base/object.cc
DECLARE_bool(is_serialize_point_cloud);

/// obstacle/onboard/hdmap_input.cc
DECLARE_double(map_radius);
DECLARE_int32(map_sample_step);
```

perception/common/perception_gflags.cc

```c++
DEFINE_string(obstacle_module_name, "perception_obstacle",
              "perception obstacle module name");
DEFINE_bool(enable_visualization, false, "enable visualization for debug");

/// obstacle/perception.cc
DEFINE_string(dag_config_path, "./conf/dag_streaming.config",
              "Onboard DAG Streaming config.");
```

## perception/conf

这里是一些配置文件，包括模块的DAG配置、各个功能所需要的配置文件的路径、不同功能之间共享数据的相关配置等。

perception/conf/config_manager.config

```
model_config_path: "model/tracker.config"
model_config_path: "model/cnn_segmentation.config"
model_config_path: "model/hdmap_roi_filter.config"
model_config_path: "model/modest_radar_detector.config"
model_config_path: "model/probabilistic_fusion.config"
model_config_path: "model/sequence_type_fuser.config"
model_config_path: "model/traffic_light/multi_camera_projection.config"
model_config_path: "model/traffic_light/recognizer.config"
model_config_path: "model/traffic_light/rectifier.config"
model_config_path: "model/traffic_light/reviser.config"
model_config_path: "model/traffic_light/preprocessor.config"
model_config_path: "model/traffic_light/subnodes.config"
```
这些配置文件会通过config_manager统一进行管理


## perception/data

运行demo所需要的数据

## perception/lib

基础库，包括了线程、时间、锁等的实现，还有对配置文件进行管理的config_manager

## perception/model

这里是机器学习所需要的模型以及权重文件，在目前的代码中来看机器学习有三个作用，一是用于对摄像机所拍摄图片中的交通信号灯的识别；二是对激光雷达的点云数据的分割，检测和划分障碍物；三是障碍物预测和聚类。官方对于3d障碍物识别的详细介绍在[https://github.com/ApolloAuto/apollo/blob/master/docs/specs/3d_obstacle_perception_cn.md](https://github.com/ApolloAuto/apollo/blob/master/docs/specs/3d_obstacle_perception_cn.md)

## perception/obstacle

本模块的主要功能之一：识别障碍物。perception/obstacle/base/object.h中有障碍物相关信息的定义

```c++
//物体坐标
  Eigen::Vector3d direction = Eigen::Vector3d(1, 0, 0);
  // the yaw angle, theta = 0.0 <=> direction = (1, 0, 0)
//物体角度
  double theta = 0.0;
  // ground center of the object (cx, cy, z_min)
  Eigen::Vector3d center = Eigen::Vector3d::Zero();
  // size of the oriented bbox, length is the size in the main direction
//物体的长宽高
  double length = 0.0;
  double width = 0.0;
  double height = 0.0;

  // tracking information
  int track_id = 0;
//物体移动速度
  Eigen::Vector3d velocity = Eigen::Vector3d::Zero();
```

在perception/obstacle/base/types.h中则有障碍物分类的定义

```c++
//障碍物类型
enum ObjectType {
  UNKNOWN = 0,
  UNKNOWN_MOVABLE = 1,
  UNKNOWN_UNMOVABLE = 2,
  PEDESTRIAN = 3,
  BICYCLE = 4,
  VEHICLE = 5,
  MAX_OBJECT_TYPE = 6,
};

enum SensorType {
  VELODYNE_64 = 0,
  VELODYNE_16 = 1,
  //这里可以看到百度可能在未来会采用成本更低的16线激光雷达
  RADAR = 2,
  CAMERA = 3,
  UNKNOWN_SENSOR_TYPE = 10,
};

enum ScoreType {
  UNKNOWN_SCORE_TYPE = 0,
  SCORE_CNN = 1,
  SCORE_RADAR = 2,
};
```

障碍物识别这个功能从Lidar和Radar两种传感器上采集数据，这两种数据对应的处理函数则是DAG图上的子节点，它们将各自输出障碍物的相关信息（前面提到的物体的三维、角度、速度、类型、是否为背景物体等）到fusion节点进行数据融合，代码里称之为概率融合probabilistic_fusion。

perception/obstacle/fusion/probabilistic_fusion/probabilistic_fusion.cc

```c++
// 1. collect sensor objects data
    for (size_t i = 0; i < multi_sensor_objects.size(); ++i) {
    .....
// 2.query related sensor frames for fusion
    sensor_manager_->GetLatestFrames(fusion_time, &frames);
    sensor_data_rw_mutex_.unlock();
    .....
// 3.peform fusion on related frames
    for (size_t i = 0; i < frames.size(); ++i) {
      FuseFrame(frames[i]);
    }
    .....
// 4.collect results
    CollectFusedObjects(fusion_time, fused_objects);
    .....
    .....
    .....
    .....
    .....
void ProbabilisticFusion::FuseFrame(const PbfSensorFramePtr &frame) {
  AINFO << "Fusing frame: " << frame->sensor_id << ","
        << "object_number: " << frame->objects.size() << ","
        << "timestamp: " << std::fixed << std::setprecision(12)
        << frame->timestamp;
  std::vector<PbfSensorObjectPtr> &objects = frame->objects;
  std::vector<PbfSensorObjectPtr> background_objects;
  std::vector<PbfSensorObjectPtr> foreground_objects;
  DecomposeFrameObjects(objects, &foreground_objects, &background_objects);

  Eigen::Vector3d ref_point = frame->sensor2world_pose.topRightCorner(3, 1);
  FuseForegroundObjects(&foreground_objects, ref_point, frame->sensor_type,
                        frame->sensor_id, frame->timestamp);
  track_manager_->RemoveLostTracks();
}
```

可以看出fusion按frame为单位进行融合，先调用DecomposeFrameObjects将物体分解为背景物体和前景物体，然后调用FuseForegroundObjects对前景物体进行融合，而在FuseForegroundObjects中主要采用的是匹配的方法

```c++
matcher_->Match(tracks, *foreground_objects, options, &assignments,
                  &unassigned_tracks, &unassigned_objects,
                  &track2measurements_dist, &measurement2tracks_dist);
```

它的具体实现在perception/obstacle/fusion/probabilistic_fusion/pbf_hm_track_object_matcher.cc中。

## perception/onboard

该目录下是一些基础功能的实现，包括DAG子节点，共享数据管理等

## perception/proto

由于整个apollo项目中共享数据都采用了protobuf序列化，所以相应的在每个模块下都有该模块所需的protobuf定义文件。在本模块中目前就三个定义文件：地图目标区域的perception_map_roi.proto、障碍物相关的perception_obstacle.proto、交通信号灯相关的traffic_light_detection.proto

## perception/tool

主要是传感器数据的读取与转化

## perception/traffic_light

信号灯识别在官方文档[https://github.com/ApolloAuto/apollo/blob/master/docs/specs/traffic_light.md](https://github.com/ApolloAuto/apollo/blob/master/docs/specs/traffic_light.md)中有详细介绍，按文档所说，信号灯识别分为预处理和处理两个阶段。

在预处理阶段主要做三件事：ROI投影、相机选择、图像同步

perception/traffic_light/onboard/tl_preprocessor_subnode.h

```c++
void CameraSelection(double ts);
bool VerifyLightsProjection(std::shared_ptr<ImageLights> image_lights);

```

相机选择函数在perception/traffic_light/preprocessor/tl_preprocessor.cc

```c++
void TLPreprocessor::SelectImage(const CarPose &pose,
                                 const LightsArray &lights_on_image_array,
                                 const LightsArray &lights_outside_image_array,
                                 CameraId *selection)
```

它是结合高精度地图和定位等信息，选择可以看到完整的信号灯的焦距最长的相机。

ROI投影的实现在perception/traffic_light/projection下，分为单个图像和多摄像头图像两种实现情况。

perception/traffic_light/onboard/tl_proc_subnode.cc

```c++
bool TLProcSubnode::InitInternal() {
  RegisterFactoryUnityRectify();
  RegisterFactoryUnityRecognize();
  RegisterFactoryColorReviser();
  .....
```

处理过程则有三个步骤：一是Rectifier，利用基于caffe的卷积神经网络的深度学习在图像的目标区域中划定信号灯的轮廓、二是Recognizer，同样是利用caffe来识别信号灯的颜色和图像、三是Reviser，通过序列、时间的因素确定最终的信号灯状态。

## 后记

从apollo发布到现在，百度将其从一个空架子逐渐搭建起了一个完整的模样，仅perception这一部分就已经有了足够大的工程量，本文的源码阅读仅仅只是从一个粗糙的宽泛的角度去试图了解apollo自动驾驶系统各个模块的相关情况，不免会有很多遗漏。

