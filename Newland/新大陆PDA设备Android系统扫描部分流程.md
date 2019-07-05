# **新大陆PDA设备Android系统扫描部分流程**

------------------------------

## 一、**扫描部分*

### 1.打开后置摄像头流程
```plantuml
participant 新大陆nlservice服务
participant 系统Camera服务
participant 系统Scanner服务
Note over 系统Camera服务: 发送Camera使用请求
系统Camera服务->新大陆nlservice服务: 发送Camera使用请求
Note over 新大陆nlservice服务: 判断当前是否有其他Camera或Scanner操作，设置标志
新大陆nlservice服务-->>系统Camera服务: 返回Success
Note over 系统Camera服务: 打开Camera操作
系统Camera服务->新大陆nlservice服务: 关闭Camera使用请求
Note over 新大陆nlservice服务: 清除Camera使用标志
新大陆nlservice服务-->>系统Scanner服务: 通知Camer使用关闭
```
### 2.打开扫描流程
```plantuml
participant 新大陆nlservice服务
participant 系统Camera服务
participant 系统Scanner服务
Note over 系统Scanner服务: 发送Camera使用请求
系统Scanner服务->新大陆nlservice服务: 发送Camera使用请求
Note over 新大陆nlservice服务: 判断当前是否有其他Camera或Scanner操作，设置标志
新大陆nlservice服务-->>系统Scanner服务: 返回Success
Note over 系统Scanner服务: 打开Camera操作
系统Scanner服务->新大陆nlservice服务: 关闭Camera使用请求
Note over 新大陆nlservice服务: 清除Camera使用标志
```

### 3.CM26扫描头操作流程
```viz
engine:dot
digraph g {
	node [shape=plaintext];
	按下扫描键 -> 调用Camera打开接口;
	调用Camera打开接口 -> nlservice [label=查询并设置使用标志];
	nlservice -> 允许打开扫描头 [label=yes];
  nlservice -> 不允许打开扫描头 [label=no];
  不允许打开扫描头 -> 扫描结束;
  允许打开扫描头 -> 进入扫描流程;
  进入扫描流程 -> 获取扫描结果;
  获取扫描结果 -> 扫描结束;
	扫描结束 -> nlservice [label=释放使用标志];
}
```

### 4.N6603扫描头操作流程
```viz
engine:dot
digraph g {
	node [shape=plaintext];
  系统Scanner服务 ->初始化N6603扫描;
  初始化N6603扫描 -> 调用Camera打开接口 [label=设置prop打开Camera个数限制];
  调用Camera打开接口 -> 进入扫描流程
  进入扫描流程 -> 获取扫描结果;
  获取扫描结果 -> 扫描结束;

  系统Camera打开接口 -> nlservice [label=查询并设置使用标志];
	nlservice -> 允许打开Camera [label=yes];
  nlservice -> 不允许打开Camera [label=no];
  不允许打开Camera -> 退出App;
  允许打开Camera -> N6603释放扫描 [label=通知释放N6603通道];
  N6603释放扫描 -> CameraApp;
  CameraApp -> 退出App;
  退出App -> nlservice [label=清除使用标志];
  nlservice -> 初始化N6603扫描;
}
```

## 二、**Nlservice部分*
```viz
engine:dot
digraph g {
	node [shape=plaintext];
  Nlservice ->CameraServiceLocalSocket;
  Nlservice ->Prop设置;
	Nlservice ->修改CPU、ROM、RAM;
	Prop设置 -> 完整设置Prop;
	Prop设置 -> 部分设置Prop [label=替换Prop中的某些字符];
}
```
