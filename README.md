# Projeto ROS Noetic — Simulação de Robô Garçom Autônomo

Este projeto tem como objetivo simular um robô garçom em ambiente ROS Noetic, utilizando Gazebo e RViz. O robô parte de uma região de cozinha e deve navegar até pontos próximos às mesas, utilizando sensores simulados, mapa do ambiente, localização por AMCL e navegação autônoma com `move_base`.

O projeto foi desenvolvido em ambiente Docker com ROS Noetic.

---

## 1. Objetivo do projeto

O sistema simulado representa um robô garçom capaz de:

* operar em um ambiente de restaurante no Gazebo;
* utilizar um sensor laser 2D para percepção;
* gerar mapa do ambiente com `gmapping`;
* salvar o mapa com `map_server`;
* localizar-se no mapa usando `AMCL`;
* receber metas pelo RViz;
* planejar trajetórias com `move_base`;
* desviar de obstáculos fixos do ambiente.

A estrutura foi pensada para futuramente permitir adaptação para o robô Pioneer P3DX.

---

## 2. Estrutura geral do sistema

```text
Gazebo
  ↓
Robô diferencial simulado
  ↓
Sensor laser /scan
  ↓
Odometria /odom
  ↓
GMapping ou AMCL
  ↓
Mapa /map
  ↓
move_base
  ↓
Comando /cmd_vel
```

Durante o mapeamento:

```text
Gazebo + robô + teleop + gmapping → mapa salvo
```

Durante a navegação:

```text
Gazebo + mapa salvo + AMCL + move_base → navegação autônoma
```

---

## 3. Criação do container Docker

O projeto foi usado com a imagem:

```bash
osrf/ros:noetic-desktop
```

Exemplo de criação do container com suporte gráfico:

```bash
docker run -it --name ros-garcom-noetic --privileged --network host -e DISPLAY=:0 -e QT_X11_NO_MITSHM=1 -e LIBGL_ALWAYS_SOFTWARE=1 -v /tmp/.X11-unix:/tmp/.X11-unix:rw osrf/ros:noetic-desktop bash
```

Para entrar novamente no container:

```bash
docker start -ai ros-garcom-noetic
```

Para abrir outro terminal dentro do mesmo container:

```bash
docker exec -it ros-garcom-noetic bash
```

Para parar o container:

```bash
docker stop ros-garcom-noetic
```

---

## 4. Instalação dos pacotes necessários

Dentro do container, rode:

```bash
apt update
apt install -y git nano python3-pip python3-catkin-tools ros-noetic-gazebo-ros-pkgs ros-noetic-gazebo-ros-control ros-noetic-navigation ros-noetic-map-server ros-noetic-amcl ros-noetic-slam-gmapping ros-noetic-teleop-twist-keyboard ros-noetic-xacro ros-noetic-robot-state-publisher ros-noetic-joint-state-publisher ros-noetic-joint-state-publisher-gui ros-noetic-tf ros-noetic-tf2-ros
```

Configure o ambiente:

```bash
echo 'source /opt/ros/noetic/setup.bash' >> ~/.bashrc
echo 'export DISPLAY=:0' >> ~/.bashrc
echo 'export QT_X11_NO_MITSHM=1' >> ~/.bashrc
echo 'export LIBGL_ALWAYS_SOFTWARE=1' >> ~/.bashrc
source ~/.bashrc
```

---

## 5. Criação do workspace

```bash
mkdir -p ~/garcom_ws/src
cd ~/garcom_ws
catkin_make
echo 'source ~/garcom_ws/devel/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

---

## 6. Criação dos pacotes

Dentro da pasta `src`:

```bash
cd ~/garcom_ws/src

catkin_create_pkg garcom_description rospy roscpp urdf xacro
catkin_create_pkg garcom_gazebo rospy roscpp gazebo_ros
catkin_create_pkg garcom_maps rospy roscpp map_server
catkin_create_pkg garcom_navigation rospy roscpp move_base amcl map_server
catkin_create_pkg garcom_missions rospy actionlib move_base_msgs geometry_msgs
```

Compile:

```bash
cd ~/garcom_ws
catkin_make
source ~/.bashrc
```

---

## 7. Estrutura recomendada do repositório

```text
garcom_ws/
└── src/
    ├── garcom_description/
    │   ├── urdf/
    │   │   └── garcom_robot.urdf.xacro
    │   └── launch/
    │
    ├── garcom_gazebo/
    │   ├── launch/
    │   │   └── restaurante_garcom.launch
    │   └── worlds/
    │       └── restaurante_simples.world
    │
    ├── garcom_maps/
    │   ├── launch/
    │   │   └── gmapping.launch
    │   └── maps/
    │       ├── restaurante.pgm
    │       └── restaurante.yaml
    │
    ├── garcom_navigation/
    │   ├── launch/
    │   │   ├── amcl.launch
    │   │   ├── move_base.launch
    │   │   └── navigation.launch
    │   └── config/
    │       ├── costmap_common_params.yaml
    │       ├── global_costmap_params.yaml
    │       ├── local_costmap_params.yaml
    │       ├── base_local_planner_params.yaml
    │       └── move_base_params.yaml
    │
    └── garcom_missions/
        └── scripts/
            └── missao_garcom.py
```

---

## 8. Arquivo `.gitignore`

Crie na raiz do repositório:

```bash
nano ~/garcom_ws/.gitignore
```

Conteúdo:

```gitignore
build/
devel/
logs/
*.pyc
__pycache__/
*.bag
.ros/
```

---

# 9. Modelo do robô

Arquivo:

```text
garcom_description/urdf/garcom_robot.urdf.xacro
```

Conteúdo:

```xml
<?xml version="1.0"?>
<robot name="garcom_robot" xmlns:xacro="http://www.ros.org/wiki/xacro">

  <xacro:property name="base_length" value="0.45"/>
  <xacro:property name="base_width"  value="0.32"/>
  <xacro:property name="base_height" value="0.18"/>

  <xacro:property name="wheel_radius" value="0.075"/>
  <xacro:property name="wheel_width"  value="0.04"/>
  <xacro:property name="wheel_y"      value="0.18"/>

  <material name="blue">
    <color rgba="0.1 0.2 0.8 1.0"/>
  </material>

  <material name="black">
    <color rgba="0.02 0.02 0.02 1.0"/>
  </material>

  <material name="gray">
    <color rgba="0.5 0.5 0.5 1.0"/>
  </material>

  <link name="base_footprint"/>

  <joint name="base_footprint_joint" type="fixed">
    <parent link="base_footprint"/>
    <child link="base_link"/>
    <origin xyz="0 0 0.12" rpy="0 0 0"/>
  </joint>

  <link name="base_link">
    <visual>
      <origin xyz="0 0 0" rpy="0 0 0"/>
      <geometry>
        <box size="${base_length} ${base_width} ${base_height}"/>
      </geometry>
      <material name="blue"/>
    </visual>

    <collision>
      <origin xyz="0 0 0" rpy="0 0 0"/>
      <geometry>
        <box size="${base_length} ${base_width} ${base_height}"/>
      </geometry>
    </collision>

    <inertial>
      <mass value="5.0"/>
      <origin xyz="0 0 0"/>
      <inertia ixx="0.08" ixy="0" ixz="0" iyy="0.10" iyz="0" izz="0.12"/>
    </inertial>
  </link>

  <link name="left_wheel_link">
    <visual>
      <origin xyz="0 0 0" rpy="1.5708 0 0"/>
      <geometry>
        <cylinder radius="${wheel_radius}" length="${wheel_width}"/>
      </geometry>
      <material name="black"/>
    </visual>

    <collision>
      <origin xyz="0 0 0" rpy="1.5708 0 0"/>
      <geometry>
        <cylinder radius="${wheel_radius}" length="${wheel_width}"/>
      </geometry>
    </collision>

    <inertial>
      <mass value="0.5"/>
      <origin xyz="0 0 0"/>
      <inertia ixx="0.001" ixy="0" ixz="0" iyy="0.001" iyz="0" izz="0.001"/>
    </inertial>
  </link>

  <joint name="left_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="left_wheel_link"/>
    <origin xyz="0 ${wheel_y} -0.045" rpy="0 0 0"/>
    <axis xyz="0 1 0"/>
  </joint>

  <link name="right_wheel_link">
    <visual>
      <origin xyz="0 0 0" rpy="1.5708 0 0"/>
      <geometry>
        <cylinder radius="${wheel_radius}" length="${wheel_width}"/>
      </geometry>
      <material name="black"/>
    </visual>

    <collision>
      <origin xyz="0 0 0" rpy="1.5708 0 0"/>
      <geometry>
        <cylinder radius="${wheel_radius}" length="${wheel_width}"/>
      </geometry>
    </collision>

    <inertial>
      <mass value="0.5"/>
      <origin xyz="0 0 0"/>
      <inertia ixx="0.001" ixy="0" ixz="0" iyy="0.001" iyz="0" izz="0.001"/>
    </inertial>
  </link>

  <joint name="right_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="right_wheel_link"/>
    <origin xyz="0 -${wheel_y} -0.045" rpy="0 0 0"/>
    <axis xyz="0 1 0"/>
  </joint>

  <link name="caster_link">
    <visual>
      <origin xyz="0 0 0"/>
      <geometry>
        <sphere radius="0.03"/>
      </geometry>
      <material name="gray"/>
    </visual>

    <collision>
      <origin xyz="0 0 0"/>
      <geometry>
        <sphere radius="0.03"/>
      </geometry>
    </collision>

    <inertial>
      <mass value="0.2"/>
      <origin xyz="0 0 0"/>
      <inertia ixx="0.0002" ixy="0" ixz="0" iyy="0.0002" iyz="0" izz="0.0002"/>
    </inertial>
  </link>

  <joint name="caster_joint" type="fixed">
    <parent link="base_link"/>
    <child link="caster_link"/>
    <origin xyz="-0.18 0 -0.09" rpy="0 0 0"/>
  </joint>

  <link name="laser_link">
    <visual>
      <origin xyz="0 0 0"/>
      <geometry>
        <cylinder radius="0.045" length="0.04"/>
      </geometry>
      <material name="gray"/>
    </visual>

    <collision>
      <origin xyz="0 0 0"/>
      <geometry>
        <cylinder radius="0.045" length="0.04"/>
      </geometry>
    </collision>

    <inertial>
      <mass value="0.1"/>
      <origin xyz="0 0 0"/>
      <inertia ixx="0.0001" ixy="0" ixz="0" iyy="0.0001" iyz="0" izz="0.0001"/>
    </inertial>
  </link>

  <joint name="laser_joint" type="fixed">
    <parent link="base_link"/>
    <child link="laser_link"/>
    <origin xyz="0.16 0 0.13" rpy="0 0 0"/>
  </joint>

  <gazebo reference="base_link">
    <material>Gazebo/Blue</material>
  </gazebo>

  <gazebo reference="left_wheel_link">
    <material>Gazebo/Black</material>
    <mu1>1.0</mu1>
    <mu2>1.0</mu2>
  </gazebo>

  <gazebo reference="right_wheel_link">
    <material>Gazebo/Black</material>
    <mu1>1.0</mu1>
    <mu2>1.0</mu2>
  </gazebo>

  <gazebo reference="caster_link">
    <material>Gazebo/Grey</material>
    <mu1>0.1</mu1>
    <mu2>0.1</mu2>
  </gazebo>

  <gazebo>
    <plugin name="differential_drive_controller" filename="libgazebo_ros_diff_drive.so">
      <robotNamespace>/</robotNamespace>

      <leftJoint>left_wheel_joint</leftJoint>
      <rightJoint>right_wheel_joint</rightJoint>

      <wheelSeparation>0.36</wheelSeparation>
      <wheelDiameter>0.15</wheelDiameter>

      <commandTopic>cmd_vel</commandTopic>
      <odometryTopic>odom</odometryTopic>
      <odometryFrame>odom</odometryFrame>
      <robotBaseFrame>base_footprint</robotBaseFrame>

      <publishWheelTF>false</publishWheelTF>
      <publishWheelJointState>true</publishWheelJointState>
      <publishTf>true</publishTf>

      <updateRate>50</updateRate>
    </plugin>
  </gazebo>

  <gazebo reference="laser_link">
    <sensor type="ray" name="laser_sensor">
      <pose>0 0 0 0 0 0</pose>
      <visualize>true</visualize>
      <update_rate>10</update_rate>

      <ray>
        <scan>
          <horizontal>
            <samples>360</samples>
            <resolution>1</resolution>
            <min_angle>-3.14159</min_angle>
            <max_angle>3.14159</max_angle>
          </horizontal>
        </scan>

        <range>
          <min>0.12</min>
          <max>8.0</max>
          <resolution>0.01</resolution>
        </range>
      </ray>

      <plugin name="gazebo_ros_laser_controller" filename="libgazebo_ros_laser.so">
        <topicName>scan</topicName>
        <frameName>laser_link</frameName>
      </plugin>
    </sensor>
  </gazebo>

</robot>
```

---

# 10. Mundo do restaurante

Arquivo:

```text
garcom_gazebo/worlds/restaurante_simples.world
```

Conteúdo:

```xml
<?xml version="1.0" ?>
<sdf version="1.6">
  <world name="restaurante_simples">

    <include>
      <uri>model://sun</uri>
    </include>

    <include>
      <uri>model://ground_plane</uri>
    </include>

    <model name="parede_superior">
      <static>true</static>
      <pose>0 3 0.5 0 0 0</pose>
      <link name="link">
        <collision name="collision">
          <geometry>
            <box>
              <size>6 0.1 1</size>
            </box>
          </geometry>
        </collision>
        <visual name="visual">
          <geometry>
            <box>
              <size>6 0.1 1</size>
            </box>
          </geometry>
        </visual>
      </link>
    </model>

    <model name="parede_inferior">
      <static>true</static>
      <pose>0 -3 0.5 0 0 0</pose>
      <link name="link">
        <collision name="collision">
          <geometry>
            <box>
              <size>6 0.1 1</size>
            </box>
          </geometry>
        </collision>
        <visual name="visual">
          <geometry>
            <box>
              <size>6 0.1 1</size>
            </box>
          </geometry>
        </visual>
      </link>
    </model>

    <model name="parede_esquerda">
      <static>true</static>
      <pose>-3 0 0.5 0 0 0</pose>
      <link name="link">
        <collision name="collision">
          <geometry>
            <box>
              <size>0.1 6 1</size>
            </box>
          </geometry>
        </collision>
        <visual name="visual">
          <geometry>
            <box>
              <size>0.1 6 1</size>
            </box>
          </geometry>
        </visual>
      </link>
    </model>

    <model name="parede_direita">
      <static>true</static>
      <pose>3 0 0.5 0 0 0</pose>
      <link name="link">
        <collision name="collision">
          <geometry>
            <box>
              <size>0.1 6 1</size>
            </box>
          </geometry>
        </collision>
        <visual name="visual">
          <geometry>
            <box>
              <size>0.1 6 1</size>
            </box>
          </geometry>
        </visual>
      </link>
    </model>

    <model name="cozinha">
      <static>true</static>
      <pose>-2.2 0 0.15 0 0 0</pose>
      <link name="link">
        <collision name="collision">
          <geometry>
            <box>
              <size>0.8 1.2 0.3</size>
            </box>
          </geometry>
        </collision>
        <visual name="visual">
          <geometry>
            <box>
              <size>0.8 1.2 0.3</size>
            </box>
          </geometry>
          <material>
            <ambient>0.7 0.4 0.2 1</ambient>
            <diffuse>0.7 0.4 0.2 1</diffuse>
          </material>
        </visual>
      </link>
    </model>

    <model name="mesa_1">
      <static>true</static>
      <pose>1.5 1.5 0.25 0 0 0</pose>
      <link name="link">
        <collision name="collision">
          <geometry>
            <cylinder>
              <radius>0.35</radius>
              <length>0.5</length>
            </cylinder>
          </geometry>
        </collision>
        <visual name="visual">
          <geometry>
            <cylinder>
              <radius>0.35</radius>
              <length>0.5</length>
            </cylinder>
          </geometry>
        </visual>
      </link>
    </model>

    <model name="mesa_2">
      <static>true</static>
      <pose>1.5 0 0.25 0 0 0</pose>
      <link name="link">
        <collision name="collision">
          <geometry>
            <cylinder>
              <radius>0.35</radius>
              <length>0.5</length>
            </cylinder>
          </geometry>
        </collision>
        <visual name="visual">
          <geometry>
            <cylinder>
              <radius>0.35</radius>
              <length>0.5</length>
            </cylinder>
          </geometry>
        </visual>
      </link>
    </model>

    <model name="mesa_3">
      <static>true</static>
      <pose>1.5 -1.5 0.25 0 0 0</pose>
      <link name="link">
        <collision name="collision">
          <geometry>
            <cylinder>
              <radius>0.35</radius>
              <length>0.5</length>
            </cylinder>
          </geometry>
        </collision>
        <visual name="visual">
          <geometry>
            <cylinder>
              <radius>0.35</radius>
              <length>0.5</length>
            </cylinder>
          </geometry>
        </visual>
      </link>
    </model>

  </world>
</sdf>
```

---

# 11. Launch do Gazebo com o robô

Arquivo:

```text
garcom_gazebo/launch/restaurante_garcom.launch
```

Conteúdo:

```xml
<launch>

  <arg name="x" default="-2.2"/>
  <arg name="y" default="-1.8"/>
  <arg name="z" default="0.05"/>

  <param name="robot_description"
         command="$(find xacro)/xacro $(find garcom_description)/urdf/garcom_robot.urdf.xacro" />

  <include file="$(find gazebo_ros)/launch/empty_world.launch">
    <arg name="world_name" value="$(find garcom_gazebo)/worlds/restaurante_simples.world"/>
    <arg name="paused" value="false"/>
    <arg name="use_sim_time" value="true"/>
    <arg name="gui" value="true"/>
  </include>

  <node name="spawn_garcom_robot"
        pkg="gazebo_ros"
        type="spawn_model"
        output="screen"
        args="-urdf -model garcom_robot -param robot_description -x $(arg x) -y $(arg y) -z $(arg z)" />

  <node name="robot_state_publisher"
        pkg="robot_state_publisher"
        type="robot_state_publisher"
        output="screen" />

</launch>
```

---

# 12. Launch do GMapping

Arquivo:

```text
garcom_maps/launch/gmapping.launch
```

Conteúdo:

```xml
<launch>

  <param name="use_sim_time" value="true"/>

  <node pkg="gmapping" type="slam_gmapping" name="slam_gmapping" output="screen">

    <param name="base_frame" value="base_footprint"/>
    <param name="odom_frame" value="odom"/>
    <param name="map_frame" value="map"/>

    <param name="map_update_interval" value="2.0"/>

    <param name="maxUrange" value="7.5"/>
    <param name="maxRange" value="8.0"/>

    <param name="sigma" value="0.05"/>
    <param name="kernelSize" value="1"/>
    <param name="lstep" value="0.05"/>
    <param name="astep" value="0.05"/>
    <param name="iterations" value="5"/>

    <param name="linearUpdate" value="0.1"/>
    <param name="angularUpdate" value="0.1"/>
    <param name="temporalUpdate" value="1.0"/>

    <param name="particles" value="30"/>

    <remap from="scan" to="/scan"/>

  </node>

</launch>
```

---

# 13. Criando e salvando o mapa

Rodar Gazebo:

```bash
roslaunch garcom_gazebo restaurante_garcom.launch
```

Rodar GMapping:

```bash
roslaunch garcom_maps gmapping.launch
```

Rodar teleop:

```bash
rosrun teleop_twist_keyboard teleop_twist_keyboard.py
```

No RViz:

```bash
rviz
```

Configuração no RViz durante o mapeamento:

```text
Fixed Frame: map

Add:
- Map /map
- RobotModel
- LaserScan /scan
- TF
```

Depois de mapear, salve o mapa:

```bash
mkdir -p ~/garcom_ws/src/garcom_maps/maps
cd ~/garcom_ws/src/garcom_maps/maps
rosrun map_server map_saver -f restaurante
```

Isso gera:

```text
restaurante.pgm
restaurante.yaml
```

Esses dois arquivos devem ser enviados para o GitHub.

---

# 14. Launch do AMCL

Arquivo:

```text
garcom_navigation/launch/amcl.launch
```

Conteúdo:

```xml
<launch>

  <param name="use_sim_time" value="true"/>

  <arg name="map_file" default="$(find garcom_maps)/maps/restaurante.yaml"/>

  <node name="map_server"
        pkg="map_server"
        type="map_server"
        args="$(arg map_file)"
        output="screen" />

  <node pkg="amcl"
        type="amcl"
        name="amcl"
        output="screen">

    <param name="odom_frame_id" value="odom"/>
    <param name="base_frame_id" value="base_footprint"/>
    <param name="global_frame_id" value="map"/>

    <param name="scan_topic" value="scan"/>
    <param name="laser_model_type" value="likelihood_field"/>

    <param name="min_particles" value="100"/>
    <param name="max_particles" value="1000"/>
    <param name="kld_err" value="0.05"/>
    <param name="kld_z" value="0.99"/>

    <param name="update_min_d" value="0.05"/>
    <param name="update_min_a" value="0.05"/>

    <param name="laser_max_range" value="8.0"/>
    <param name="laser_min_range" value="0.12"/>
    <param name="laser_likelihood_max_dist" value="2.0"/>

    <param name="odom_model_type" value="diff"/>
    <param name="odom_alpha1" value="0.2"/>
    <param name="odom_alpha2" value="0.2"/>
    <param name="odom_alpha3" value="0.2"/>
    <param name="odom_alpha4" value="0.2"/>

    <param name="transform_tolerance" value="0.5"/>
    <param name="resample_interval" value="1"/>

  </node>

</launch>
```

---

# 15. Configurações do move_base

## 15.1. `costmap_common_params.yaml`

Arquivo:

```text
garcom_navigation/config/costmap_common_params.yaml
```

Conteúdo:

```yaml
obstacle_range: 2.5
raytrace_range: 3.0

footprint: [[0.225, 0.16], [0.225, -0.16], [-0.225, -0.16], [-0.225, 0.16]]

inflation_radius: 0.35
cost_scaling_factor: 3.0

observation_sources: laser_scan_sensor

laser_scan_sensor:
  sensor_frame: laser_link
  data_type: LaserScan
  topic: /scan
  marking: true
  clearing: true
```

---

## 15.2. `global_costmap_params.yaml`

Arquivo:

```text
garcom_navigation/config/global_costmap_params.yaml
```

Conteúdo:

```yaml
global_costmap:
  global_frame: map
  robot_base_frame: base_footprint

  update_frequency: 5.0
  publish_frequency: 2.0

  transform_tolerance: 0.5

  static_map: true
  rolling_window: false
```

---

## 15.3. `local_costmap_params.yaml`

Arquivo:

```text
garcom_navigation/config/local_costmap_params.yaml
```

Conteúdo:

```yaml
local_costmap:
  global_frame: odom
  robot_base_frame: base_footprint

  update_frequency: 10.0
  publish_frequency: 5.0

  transform_tolerance: 0.5

  static_map: false
  rolling_window: true

  width: 3.0
  height: 3.0
  resolution: 0.05
```

---

## 15.4. `base_local_planner_params.yaml`

Arquivo:

```text
garcom_navigation/config/base_local_planner_params.yaml
```

Conteúdo:

```yaml
TrajectoryPlannerROS:

  max_vel_x: 0.18
  min_vel_x: 0.02

  max_vel_theta: 0.45
  min_in_place_vel_theta: 0.12

  acc_lim_x: 0.5
  acc_lim_theta: 0.8

  holonomic_robot: false

  meter_scoring: true

  xy_goal_tolerance: 0.30
  yaw_goal_tolerance: 3.14
  latch_xy_goal_tolerance: true

  sim_time: 2.0
  sim_granularity: 0.025
  angular_sim_granularity: 0.025

  vx_samples: 10
  vtheta_samples: 40

  path_distance_bias: 0.8
  goal_distance_bias: 0.7
  occdist_scale: 0.04

  escape_vel: -0.05
```

---

## 15.5. `move_base_params.yaml`

Arquivo:

```text
garcom_navigation/config/move_base_params.yaml
```

Conteúdo:

```yaml
base_global_planner: navfn/NavfnROS
base_local_planner: base_local_planner/TrajectoryPlannerROS

controller_frequency: 10.0
planner_frequency: 2.0

planner_patience: 5.0
controller_patience: 15.0

conservative_reset_dist: 1.0

recovery_behavior_enabled: true
clearing_rotation_allowed: true
shutdown_costmaps: false

oscillation_timeout: 10.0
oscillation_distance: 0.2
```

---

# 16. Launch do move_base

Arquivo:

```text
garcom_navigation/launch/move_base.launch
```

Conteúdo:

```xml
<launch>

  <param name="use_sim_time" value="true"/>

  <node pkg="move_base"
        type="move_base"
        respawn="false"
        name="move_base"
        output="screen">

    <rosparam file="$(find garcom_navigation)/config/costmap_common_params.yaml"
              command="load"
              ns="global_costmap" />

    <rosparam file="$(find garcom_navigation)/config/costmap_common_params.yaml"
              command="load"
              ns="local_costmap" />

    <rosparam file="$(find garcom_navigation)/config/global_costmap_params.yaml"
              command="load" />

    <rosparam file="$(find garcom_navigation)/config/local_costmap_params.yaml"
              command="load" />

    <rosparam file="$(find garcom_navigation)/config/base_local_planner_params.yaml"
              command="load" />

    <rosparam file="$(find garcom_navigation)/config/move_base_params.yaml"
              command="load" />

  </node>

</launch>
```

---

# 17. Launch completo de navegação

Arquivo:

```text
garcom_navigation/launch/navigation.launch
```

Conteúdo:

```xml
<launch>

  <param name="use_sim_time" value="true"/>

  <include file="$(find garcom_navigation)/launch/amcl.launch" />

  <include file="$(find garcom_navigation)/launch/move_base.launch" />

</launch>
```

---

# 18. Rodando o sistema completo

Terminal 1:

```bash
roslaunch garcom_gazebo restaurante_garcom.launch
```

Terminal 2:

```bash
roslaunch garcom_navigation navigation.launch
```

Terminal 3:

```bash
rviz
```

No RViz, configure:

```text
Fixed Frame: map
```

Adicione:

```text
Map → /map
RobotModel
LaserScan → /scan
PoseArray → /particlecloud
TF
Path → /move_base/NavfnROS/plan
Map → /move_base/global_costmap/costmap
Map → /move_base/local_costmap/costmap
```

Antes de mandar uma meta, use:

```text
2D Pose Estimate
```

Coloque o robô no ponto correto do mapa, de acordo com sua posição no Gazebo.

Depois envie uma meta com:

```text
2D Nav Goal
```

A meta deve ser enviada para um ponto livre próximo à mesa, não para o centro da mesa.

---

# 19. Testes úteis

Listar tópicos:

```bash
rostopic list
```

Verificar odometria:

```bash
rostopic echo /odom
```

Verificar laser:

```bash
rostopic echo /scan
```

Verificar pose do AMCL:

```bash
rostopic echo /amcl_pose
```

Verificar nuvem de partículas:

```bash
rostopic echo -n 1 /particlecloud
```

Verificar transformações:

```bash
rosrun tf tf_echo map base_footprint
```

Verificar comandos de velocidade:

```bash
rostopic echo /cmd_vel
```

Limpar costmaps:

```bash
rosservice call /move_base/clear_costmaps "{}"
```

---

# 20. Script opcional de missão automática

Este script permite enviar o robô automaticamente para uma mesa e depois retornar para a cozinha. As coordenadas devem ser ajustadas conforme os valores reais obtidos no RViz ou no tópico `/amcl_pose`.

Crie a pasta:

```bash
mkdir -p ~/garcom_ws/src/garcom_missions/scripts
```

Arquivo:

```text
garcom_missions/scripts/missao_garcom.py
```

Conteúdo:

```python
#!/usr/bin/env python3

import sys
import rospy
import actionlib
from move_base_msgs.msg import MoveBaseAction, MoveBaseGoal


def criar_meta(nome, x, y, z, w):
    goal = MoveBaseGoal()
    goal.target_pose.header.frame_id = "map"
    goal.target_pose.header.stamp = rospy.Time.now()

    goal.target_pose.pose.position.x = x
    goal.target_pose.pose.position.y = y
    goal.target_pose.pose.position.z = 0.0

    goal.target_pose.pose.orientation.x = 0.0
    goal.target_pose.pose.orientation.y = 0.0
    goal.target_pose.pose.orientation.z = z
    goal.target_pose.pose.orientation.w = w

    rospy.loginfo("Meta criada: %s | x=%.2f y=%.2f", nome, x, y)
    return goal


def enviar_meta(client, nome, x, y, z, w):
    rospy.loginfo("Enviando robô para: %s", nome)

    goal = criar_meta(nome, x, y, z, w)
    client.send_goal(goal)

    finished = client.wait_for_result(rospy.Duration(120))

    if not finished:
        rospy.logwarn("Tempo excedido para chegar em: %s", nome)
        client.cancel_goal()
        return False

    state = client.get_state()

    if state == 3:
        rospy.loginfo("Robô chegou em: %s", nome)
        return True
    else:
        rospy.logwarn("Robô não conseguiu chegar em: %s | estado=%s", nome, state)
        return False


def main():
    rospy.init_node("missao_garcom")

    pontos = {
        "cozinha": {
            "x": -2.0,
            "y": -1.5,
            "z": 0.0,
            "w": 1.0
        },
        "mesa_1": {
            "x": 1.0,
            "y": 1.3,
            "z": 0.0,
            "w": 1.0
        },
        "mesa_2": {
            "x": 1.0,
            "y": 0.0,
            "z": 0.0,
            "w": 1.0
        },
        "mesa_3": {
            "x": 1.0,
            "y": -1.3,
            "z": 0.0,
            "w": 1.0
        }
    }

    if len(sys.argv) < 2:
        rospy.logerr("Uso: rosrun garcom_missions missao_garcom.py mesa_1")
        rospy.logerr("Opções: mesa_1, mesa_2, mesa_3")
        return

    mesa = sys.argv[1]

    if mesa not in pontos:
        rospy.logerr("Mesa inválida: %s", mesa)
        rospy.logerr("Opções disponíveis: mesa_1, mesa_2, mesa_3")
        return

    client = actionlib.SimpleActionClient("move_base", MoveBaseAction)

    rospy.loginfo("Aguardando servidor move_base...")
    client.wait_for_server()
    rospy.loginfo("Servidor move_base conectado.")

    destino = pontos[mesa]
    cozinha = pontos["cozinha"]

    sucesso_ida = enviar_meta(
        client,
        mesa,
        destino["x"],
        destino["y"],
        destino["z"],
        destino["w"]
    )

    if sucesso_ida:
        rospy.loginfo("Simulando entrega do pedido...")
        rospy.sleep(5.0)

        enviar_meta(
            client,
            "cozinha",
            cozinha["x"],
            cozinha["y"],
            cozinha["z"],
            cozinha["w"]
        )


if __name__ == "__main__":
    try:
        main()
    except rospy.ROSInterruptException:
        pass
```

Dê permissão de execução:

```bash
chmod +x ~/garcom_ws/src/garcom_missions/scripts/missao_garcom.py
```

Compile:

```bash
cd ~/garcom_ws
catkin_make
source ~/.bashrc
```

Uso:

```bash
rosrun garcom_missions missao_garcom.py mesa_1
```

ou:

```bash
rosrun garcom_missions missao_garcom.py mesa_2
```

ou:

```bash
rosrun garcom_missions missao_garcom.py mesa_3
```

As coordenadas do script devem ser substituídas pelos valores reais obtidos com:

```bash
rostopic echo -n 1 /amcl_pose
```

---

# 21. Ordem completa para replicar o projeto

## Etapa 1 — Criar container

```bash
docker run -it --name ros-garcom-noetic --privileged --network host -e DISPLAY=:0 -e QT_X11_NO_MITSHM=1 -e LIBGL_ALWAYS_SOFTWARE=1 -v /tmp/.X11-unix:/tmp/.X11-unix:rw osrf/ros:noetic-desktop bash
```

## Etapa 2 — Instalar dependências

```bash
apt update
apt install -y git nano python3-pip python3-catkin-tools ros-noetic-gazebo-ros-pkgs ros-noetic-gazebo-ros-control ros-noetic-navigation ros-noetic-map-server ros-noetic-amcl ros-noetic-slam-gmapping ros-noetic-teleop-twist-keyboard ros-noetic-xacro ros-noetic-robot-state-publisher ros-noetic-joint-state-publisher ros-noetic-joint-state-publisher-gui ros-noetic-tf ros-noetic-tf2-ros
```

## Etapa 3 — Criar workspace

```bash
mkdir -p ~/garcom_ws/src
cd ~/garcom_ws
catkin_make
echo 'source /opt/ros/noetic/setup.bash' >> ~/.bashrc
echo 'source ~/garcom_ws/devel/setup.bash' >> ~/.bashrc
echo 'export DISPLAY=:0' >> ~/.bashrc
echo 'export QT_X11_NO_MITSHM=1' >> ~/.bashrc
echo 'export LIBGL_ALWAYS_SOFTWARE=1' >> ~/.bashrc
source ~/.bashrc
```

## Etapa 4 — Criar pacotes

```bash
cd ~/garcom_ws/src
catkin_create_pkg garcom_description rospy roscpp urdf xacro
catkin_create_pkg garcom_gazebo rospy roscpp gazebo_ros
catkin_create_pkg garcom_maps rospy roscpp map_server
catkin_create_pkg garcom_navigation rospy roscpp move_base amcl map_server
catkin_create_pkg garcom_missions rospy actionlib move_base_msgs geometry_msgs
```

## Etapa 5 — Criar arquivos

Criar os arquivos descritos neste README:

```text
garcom_description/urdf/garcom_robot.urdf.xacro
garcom_gazebo/worlds/restaurante_simples.world
garcom_gazebo/launch/restaurante_garcom.launch
garcom_maps/launch/gmapping.launch
garcom_navigation/launch/amcl.launch
garcom_navigation/launch/move_base.launch
garcom_navigation/launch/navigation.launch
garcom_navigation/config/costmap_common_params.yaml
garcom_navigation/config/global_costmap_params.yaml
garcom_navigation/config/local_costmap_params.yaml
garcom_navigation/config/base_local_planner_params.yaml
garcom_navigation/config/move_base_params.yaml
```

## Etapa 6 — Compilar

```bash
cd ~/garcom_ws
catkin_make
source ~/.bashrc
```

## Etapa 7 — Rodar Gazebo

```bash
roslaunch garcom_gazebo restaurante_garcom.launch
```

## Etapa 8 — Mapear

```bash
roslaunch garcom_maps gmapping.launch
```

Em outro terminal:

```bash
rosrun teleop_twist_keyboard teleop_twist_keyboard.py
```

Salvar mapa:

```bash
cd ~/garcom_ws/src/garcom_maps/maps
rosrun map_server map_saver -f restaurante
```

## Etapa 9 — Navegar com AMCL e move_base

Terminal 1:

```bash
roslaunch garcom_gazebo restaurante_garcom.launch
```

Terminal 2:

```bash
roslaunch garcom_navigation navigation.launch
```

Terminal 3:

```bash
rviz
```

No RViz:

```text
Fixed Frame: map
2D Pose Estimate
2D Nav Goal
```

---

# 22. Problemas comuns

## RViz mostra erro `unknown frame map`

Solução:

* durante teleop simples, use `Fixed Frame = odom`;
* durante AMCL ou GMapping, use `Fixed Frame = map`.

## Robô aparece em local errado no mapa

Use no RViz:

```text
2D Pose Estimate
```

Clique na posição correta do robô no mapa e arraste para indicar a orientação.

## Robô fica girando no final

Ajuste no `base_local_planner_params.yaml`:

```yaml
xy_goal_tolerance: 0.30
yaw_goal_tolerance: 3.14
```

Também evite mandar meta no centro da mesa. Use pontos livres ao lado da mesa.

## Local costmap parece diferente do global costmap

Isso é normal.

O `global_costmap` usa o mapa inteiro no frame `map`.

O `local_costmap` é uma janela menor ao redor do robô no frame `odom`.

## GMapping e AMCL não devem rodar juntos

Para mapear, use:

```bash
roslaunch garcom_maps gmapping.launch
```

Para navegar com mapa pronto, use:

```bash
roslaunch garcom_navigation navigation.launch
```

Não rode os dois ao mesmo tempo.

---

# 23. Próximas melhorias

Possíveis melhorias futuras:

* adicionar obstáculos móveis no Gazebo;
* criar lógica de múltiplas entregas;
* calcular melhor ordem de atendimento das mesas;
* substituir o robô genérico pelo Pioneer P3DX;
* adicionar câmera;
* criar interface simples para seleção de mesa;
* salvar configuração do RViz em arquivo `.rviz`;
* criar launch único para simulação completa;
* adicionar vídeo demonstrativo ao README.

---

# 24. Créditos

Projeto desenvolvido para simulação acadêmica de navegação autônoma em ROS Noetic, usando Gazebo, RViz, GMapping, AMCL e move_base.
