# Solutions-for-adsorption.
The software and hardware solutions for adsorption of the adsorption-type amphibious unmanned aerial vehicle.
# 首先鸣谢谭科师兄的[paper](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=10848216)与[论文](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=10848216)提供的方案
## 系统组成部分
核心控制器：DJI C型开发板
执行单元：高性能离心风扇（pwm）
传感单元：压力传感器（ADC）、姿态传感器（内置）、电流传感器（ADC）
电源单元：电源管理电路
## 工作流程
通过姿态传感器和压力传感器监测机器人状态
基于机器人运动状态、表面特性和运动方向计算所需吸附力
自适应调节离心风扇的转速
实时监测吸附状态，确保系统安全
## 硬件选型（暂时）
压力传感器,型号：Honeywell HSCDANN005PGAA5
电流传感器,型号：ACS712-30A
## 软件架构（暂时）
```
┌───────────────────────────────────────────────────┐
│                    应用层(App Layer)               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │吸附控制应用  │  │安全管理应用  │  │  系统管理   │ │
│  │AdsorptionApp│  │ SafetyApp   │  │SystemManager│ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
└───────────────────────────────────────────────────┘
              │             │             │
              ▼             ▼             ▼
┌───────────────────────────────────────────────────┐
│                    服务层(Service Layer)           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │ 吸附力服务  │  │ 状态监测服务 │  │ 通信&日志服务│ │
│  │AdsorptionSvc│  │MonitoringSvc│  │CommLogSvc   │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
└───────────────────────────────────────────────────┘
              │             │             │
              ▼             ▼             ▼
┌───────────────────────────────────────────────────┐
│                    驱动层(Driver Layer)            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │  硬件抽象   │  │  设备驱动   │  │ 传感器驱动  │ │
│  │   HAL       │  │Device Drivers│  │Sensor Drivers│ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
└───────────────────────────────────────────────────┘
              │             │             │
              ▼             ▼             ▼
┌───────────────────────────────────────────────────┐
│                    系统层(System Layer)            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │ FreeRTOS    │  │  DJI SDK    │  │  系统服务   │ │
│  │   内核      │  │             │  │System Services│ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
└───────────────────────────────────────────────────┘
```
系统层(System Layer)：
职责：提供操作系统功能、硬件抽象和基础系统服务
组件：
FreeRTOS内核：任务调度、内存管理、中断处理
DJI SDK：DJI C型开发板的底层驱动和功能
系统服务：时钟、电源管理、错误处理

驱动层(Driver Layer)：
职责：提供对硬件设备的统一访问接口，隔离硬件细节
组件：
硬件抽象(HAL)：硬件寄存器操作封装
设备驱动：PWM、ADC、CAN、UART驱动
传感器驱动：IMU、压力传感器、电流传感器驱动

服务层(Service Layer)：
职责：提供核心功能服务，封装业务逻辑
组件：
吸附力服务：负压计算、风扇控制、PID控制器
状态监测服务：运行状态监测、异常检测、诊断
通信与日志服务：通信协议、数据记录、事件日志

应用层(App Layer)：
职责：协调各服务完成具体应用需求，实现业务流程
组件：
吸附控制应用：整合吸附力计算、表面适应策略
安全管理应用：安全状态评估、应急处理
系统管理：系统配置、资源管理、用户界面
## 控制算法设计
### 1.吸附力满足：
攀爬机器人与墙面之间的附着力满足：
$F_{adsorption} = P\cdot A$
其中：
$F_{adsorption}$是吸附力（N）
P是负压值（Pa）
A是有效吸附面积（m²）
为克服重力，必须满足：
$F_{adsorption} \geq m \cdot g\cdot k_{safety}$
其中：
m 是机器人质量（kg）
g 是重力加速度（9.81 m/s²）
$k_{safety}$ 是安全系数（通常3-5）
### 2. 外环控制器：自适应吸附力计算
#### 2.1 基础吸附力考虑机器人质量和安全系数：
$F_{base} = m \cdot g \cdot k_{safety}$
#### 2.2 姿态补偿
考虑机器人姿态(横滚角roll和俯仰角pitch)的影响：
$F_{attitude} = m \cdot g \cdot (k_{roll} \cdot |\sin(\phi_{roll})| + k_{pitch} \cdot |\sin(\phi_{pitch})|)$
其中：
$\phi_{roll}、\phi_{pitch}$ 是机器人的横滚角和俯仰角(rad)
$k_{roll}、k_{pitch}$ 是姿态补偿系数(通常0.2-0.5)
#### 2.3 运动状态补偿
考虑机器人加速度的影响：
$F_{motion} = m \cdot (k_{up} \cdot \max(0, a_y) + k_{lateral} \cdot |a_x| + k_{dynamic} \cdot (|a_{total}| - g))$
其中：
$a_x, a_y$ 是机器人在x轴(横向)和y轴(纵向)的加速度(m/s²)
$a_{total}$ 是合加速度大小(m/s²)
$k_{up}, k_{lateral}, k_{dynamic}$ 是相应的补偿系数
#### 2.4 曲率补偿
表面曲率对吸附的影响：
$F_{curvature} = m \cdot g \cdot k_{curv} \cdot \frac{1}{R}$
其中：
R 是表面曲率半径(m)
$k_{curv}$ 是曲率补偿系数(通常1.0-2.0)
#### 2.5 总吸附力与目标负压计算
$F_{total} = F_{base} + F_{attitude} + F_{motion} + F_{curvature}$
目标负压：
$P_{target} = -\frac{F_{total}}{n \cdot A_{chamber}}$
其中：
n 是吸附腔数量
$A_{chamber}$ 是单个吸附腔有效面积(m²)
负号表示负压
#### 2.6 算法流程
读取传感器数据(姿态、加速度、曲率)
计算基础吸附力
计算姿态补偿力
计算运动状态补偿力
计算曲率补偿力
计算总吸附力
计算目标负压值
将目标负压传递给内环控制器
### 3. 内环控制器：增强型PID负压控制
#### 3.1 经典PID控制原理
标准PID控制器的控制律为：
$u(t) = K_p \cdot e(t) + K_i \cdot \int_{0}^{t} e(\tau) d\tau + K_d \cdot \frac{de(t)}{dt}$
其中：
u(t) 是控制输出
e(t) 是误差信号：e(t) = r(t) - y(t)
r(t) 是参考输入(目标负压)
y(t) 是系统输出(实际负压)
$K_p, K_i, K_d$ 分别是比例、积分、微分系数
#### 3.2 增强型PID控制策略
为提高负压控制性能，我们对标准PID进行以下增强：
#### 3.2.1 积分分离
- 积分分离减小大误差下的积分作用，防止积分饱和：
$K_{i_effective} = K_i \cdot (1 - \textrm{sat}(\frac{|e(t)|}{E_{max}}))$
其中：
$E_{max}$ 是误差阈值
$\textrm{sat}()$ 是饱和函数
#### 3.2.2 微分先行
- 对测量值而非误差求微分，减小设定值变化带来的微分冲击：
$D_{term} = -K_d \cdot \frac{dy(t)}{dt} \quad \textrm{(而非)} \quad K_d \cdot \frac{de(t)}{dt}$
#### 3.2.3 变速积分
- 误差越大，积分作用越小：
$I_{term} = K_i \cdot \int_{0}^{t} e(\tau) \cdot \frac{E_{max} - |e(\tau)|}{E_{max}} d\tau$
#### 3.2.4 死区补偿
（未写）
#### 3.2.5 前馈补偿
- 添加前馈项加速响应：
$u(t) = u_{PID}(t) + K_{ff} \cdot r(t)$
- 其中K_{ff}是前馈系数。
#### 3.3 自适应PID参数调整
- PID参数根据工作状态动态调整：
$K_p = K_{p_base} \cdot f_p(P, \textrm{state})$
$K_i = K_{i_base} \cdot f_i(P, \textrm{state})$
$K_d = K_{d_base} \cdot f_d(P, \textrm{state})$
- 其中 $f_p$, $f_i$, $f_d$是根据当前负压值P和系统状态 $\textrm{state}$的调整函数。
#### 3.4 算法流程

- 接收目标负压值
- 读取当前负压值
- 计算误差
- 根据运行状态调整PID参数
- 计算比例项
- 计算积分项(应用积分分离和变速积分)
- 计算微分项(应用微分先行)
- 添加前馈项
- 应用死区补偿
- 输出限幅
- 生成风扇PWM信号
# Task for new member
## TASK1：完成传感器选型、购买，等待期间完成风机的硬件驱动层（将hal的pwm封装成风机的上下限度），学会利用已经有的c板陀螺仪读取数据姿态结算（此部分代码写好了）
## TASK2：传感器回来了，完成未完成的TASK1，利用ADC（取决于硬件）读取传感器数据，得到数据后进行控制外环——风机的PID（你也可以提出更好的算法）
## TASK3：完成控制内环
## TASK4：启动、紧急叫停报错等程序解决



