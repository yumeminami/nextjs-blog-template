---
title: 如何使用 ros1 bridge
date: 2025-05-17T21:10:00+08:00
updated: 2025-05-17T21:10:00+08:00
keywords: ["ros1", "bridge", "log"]
featured: true
summary: "如何使用 ros1 bridge 以及为修复了官方仓库的bug"
---

## 项目地址

https://github.com/ros2/ros1_bridge


## 背景

在公司项目中，本身的机器人项目是基于 `ros1` 的，有客户本身的自有的工具或者项目是基于 `ros2` 的，这就不便于客户使用了。

在满足客户方便使用的需求下，通过评估再实现一套 `ros2` 的机器人程序的工作量下，不如通过走 `ros` 桥接去实现通信更简单。

## 接口文件格式化

在使用 `ros1_bridge` 过程中，发现原来 `ROS2` 对 `interface` 的 definition 要求这么严格， 需要满足以下要求：

- `msg` 或者 `srv` 的命名需要满足 CamelCase 的命名规范
- `interface` 文件中的所有 field 字段需要满足 snake_case 的命名规范
- `interface` 文件中的所有宏定义需要全大写

然而 `ROS1` 的 `interface` 的 definition 并没有这么严格，因此项目内部的接口定义就比较 casual，导致不能直接修改 `CMakeLists.txt` 就能够编译出 `ros2` 的 `msg` 和 `srv` 。

所以我实现了一个小 tool 用于将所有的接口定义都变成满足 `ROS2` 的 `interface` 标准。

```python
#!/usr/bin/env python3
import os
import re

def camel_to_snake(name):
    """将 camelCase 或 PascalCase 转换为 snake_case，并确保首字母小写"""
    # 先处理第一个单词
    first_word = name[0].lower() + name[1:]
    # 然后处理剩余的驼峰
    s1 = re.sub('(.)([A-Z][a-z]+)', r'\1_\2', first_word)
    s2 = re.sub('([a-z0-9])([A-Z])', r'\1_\2', s1)
    return s2.lower()

def capitalize_type(type_name):
    """确保自定义消息类型首字母大写，但保持基本类型的原始大小写"""
    # 基本类型保持原始大小写
    basic_types = {
        'bool', 'byte', 'char',
        'float32', 'float64',
        'int8', 'uint8',
        'int16', 'uint16',
        'int32', 'uint32',
        'int64', 'uint64',
        'string', 'wstring'
    }
    
    # 处理数组标记
    array_suffix = ''
    # 匹配 [] 或 [数字]
    array_match = re.search(r'(\[\d*\])', type_name)
    if array_match:
        array_suffix = array_match.group(1)
        type_name = type_name.replace(array_suffix, '')
    
    # 如果是基本类型，统一转换为小写
    if type_name.lower() in {t.lower() for t in basic_types}:
        return type_name.lower() + array_suffix
    
    # 处理可能包含命名空间的类型
    parts = type_name.split('/')
    if len(parts) > 1:
        # 保持命名空间小写，类型名首字母大写
        return '/'.join(parts[:-1] + [parts[-1][0].upper() + parts[-1][1:]]) + array_suffix
    
    # 自定义类型首字母大写
    return type_name[0].upper() + type_name[1:] + array_suffix

def convert_fields_to_snake_case(file_path):
    """转换文件中的字段名为 snake_case 并确保类型名称正确"""
    with open(file_path, 'r') as f:
        content = f.read()
    
    modified = False
    new_lines = []
    
    for line in content.splitlines():
        # 跳过注释和空行
        if not line.strip() or line.strip().startswith('#'):
            new_lines.append(line)
            continue
        
        # 修正 Header 和 Time 类型
        if 'Header' in line and not 'std_msgs/Header' in line:
            line = line.replace('Header', 'std_msgs/Header')
            modified = True
        elif 'Time' in line and not 'builtin_interfaces/Time' in line and not line.strip().startswith('#'):
            line = line.replace('Time', 'builtin_interfaces/Time')
            modified = True
        elif re.search(r'\btime\b', line, re.IGNORECASE) and not 'builtin_interfaces/Time' in line and not line.strip().startswith('#'):
            line = re.sub(r'\btime\b', 'builtin_interfaces/Time', line, flags=re.IGNORECASE)
            modified = True
        
        # 处理字段定义
        # 使用正则表达式匹配，以处理多个空格的情况
        match = re.match(r'^(\s*)([a-zA-Z0-9_/]+(?:\[\d*\])?)\s+([a-zA-Z][a-zA-Z0-9_]*)(.*?)$', line.strip())
        if match:
            indent = line[:len(line) - len(line.lstrip())]
            type_part = match.group(2)
            field_name = match.group(3)
            rest = match.group(4)
            
            # 检查是否是常量定义
            if '=' in line:
                if not field_name.isupper():  # 如果常量名不是全大写
                    new_const = field_name.upper()
                    line = f"{indent}{type_part} {new_const}{rest}"
                    modified = True
            else:
                # 处理类型名（包括数组）
                new_type = capitalize_type(type_part)
                # 处理字段名
                new_field = camel_to_snake(field_name)
                
                if new_type != type_part or new_field != field_name:
                    modified = True
                    line = f"{indent}{new_type} {new_field}{rest}"
        
        new_lines.append(line)
    
    if modified:
        with open(file_path, 'w') as f:
            f.write('\n'.join(new_lines) + '\n')
        print(f"Modified fields in: {file_path}")

def capitalize_first_letter(directory):
    """处理目录中的所有消息文件"""
    if not os.path.exists(directory):
        print(f"Directory {directory} does not exist")
        return
    
    for filename in os.listdir(directory):
        if not (filename.endswith('.msg') or filename.endswith('.srv')):
            continue
            
        # 只将第一个字母大写
        new_filename = filename[0].upper() + filename[1:]
        
        file_path = os.path.join(directory, filename)
        new_path = os.path.join(directory, new_filename)
        
        # 首先转换文件中的字段名和类型名
        convert_fields_to_snake_case(file_path)
        
        # 如果文件名需要改变，则重命名
        if filename != new_filename:
            try:
                os.rename(file_path, new_path)
                print(f"Renamed: {filename} -> {new_filename}")
            except Exception as e:
                print(f"Error renaming {filename}: {str(e)}")

def main():
    script_dir = os.path.dirname(os.path.abspath(__file__))
    
    
    msg_dir = os.path.join(script_dir, 'msg')
    print("\nProcessing msg directory...")
    capitalize_first_letter(msg_dir)
    
    srv_dir = os.path.join(script_dir, 'srv')
    print("\nProcessing srv directory...")
    capitalize_first_letter(srv_dir)

if __name__ == "__main__":
    main()
```

使用方法：

将此 `python` 脚本放在您的 `ros1_msgs` 的根目录下，然后执行 `python3 convert_interface.py` 即可。

```bash
cd <your_ros1_ws>/src/ros1_msgs
python3 convert_interface.py
```

## 生成规则映射

在 `ros1_bridge` 中，默认仅支持 `ros1` 和 `ros2` 官方自带的接口，例如 `std_msgs`, `geometry_msgs`, `sensor_msgs` 等。

如果需要支持 `customized` 的 `interface` ，则需要手动生成 `rule` 映射文件。

官方提供了映射规则文件的语法以及如何配置: https://github.com/ros2/ros1_bridge/blob/master/doc/index.rst

这个时候我也写了个小 tool 用于生成 `rule` 映射文件(因为仓库的接口文件定义太多了，逐个手写得写到猴年马月，而且还有可能漏掉，而且以后更新了接口文件，还得手动维护)。

```python
#!/usr/bin/env python3
import os
import sys
import yaml
import argparse
from typing import Dict, List, Tuple, Optional
import re

def camel_to_snake(name: str) -> str:
    """Convert camelCase or PascalCase to snake_case"""
    first_word = name[0].lower() + name[1:]
    s1 = re.sub('(.)([A-Z][a-z]+)', r'\1_\2', first_word)
    s2 = re.sub('([a-z0-9])([A-Z])', r'\1_\2', s1)
    return s2.lower()

def capitalize_type(type_name: str) -> str:
    """Ensure custom message types start with capital letter"""
    basic_types = {
        'bool', 'byte', 'char',
        'float32', 'float64',
        'int8', 'uint8',
        'int16', 'uint16',
        'int32', 'uint32',
        'int64', 'uint64',
        'string', 'wstring'
    }
    
    array_suffix = ''
    array_match = re.search(r'(\[\d*\])', type_name)
    if array_match:
        array_suffix = array_match.group(1)
        type_name = type_name.replace(array_suffix, '')
    
    if type_name.lower() in {t.lower() for t in basic_types}:
        return type_name.lower() + array_suffix
    
    parts = type_name.split('/')
    if len(parts) > 1:
        return '/'.join(parts[:-1] + [parts[-1][0].upper() + parts[-1][1:]]) + array_suffix
    
    return type_name[0].upper() + type_name[1:] + array_suffix

def parse_message_file(file_path: str) -> List[Tuple[str, str]]:
    """Parse a message file and return list of (type, field_name) tuples"""
    fields = []
    with open(file_path, 'r') as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith('#'):
                continue
            
            # Match field definition
            match = re.match(r'^([a-zA-Z0-9_/]+(?:\[\d*\])?)\s+([a-zA-Z][a-zA-Z0-9_]*)(.*?)$', line)
            if match:
                type_part = match.group(1)
                field_name = match.group(2)
                
                # Skip constant definitions
                if '=' in line:
                    continue
                
                # Process type and field name
                new_type = capitalize_type(type_part)
                fields.append((new_type, field_name))  # Keep original field name
    
    return fields

def compare_messages(ros1_fields: List[Tuple[str, str]], 
                    ros2_fields: List[Tuple[str, str]]) -> Optional[Dict[str, str]]:
    """Compare message fields and return field mapping if different"""
    if len(ros1_fields) != len(ros2_fields):
        return {ros1_field: ros2_field for (_, ros1_field), (_, ros2_field) in zip(ros1_fields, ros2_fields)}
    
    field_mapping = {}
    for (ros1_type, ros1_field), (ros2_type, ros2_field) in zip(ros1_fields, ros2_fields):
        # 检查字段名是否符合 snake_case 规则
        ros1_snake = camel_to_snake(ros1_field)
        ros2_snake = camel_to_snake(ros2_field)
        
        # 如果原始字段名不符合 snake_case 规则，添加到映射中
        if ros1_field != ros1_snake or ros2_field != ros2_snake:
            field_mapping[ros1_field] = ros2_field
    
    return field_mapping if field_mapping else None

def find_message_files(package_path: str) -> List[str]:
    """Find all .msg files in the package"""
    msg_files = []
    msg_dir = os.path.join(package_path, 'msg')
    if os.path.exists(msg_dir):
        for file in os.listdir(msg_dir):
            if file.endswith('.msg'):
                msg_files.append(os.path.join(msg_dir, file))
    return msg_files

def get_message_name(file_path: str) -> str:
    """Get message name without extension and in lowercase for comparison"""
    return os.path.basename(file_path)[:-4].lower()

def parse_service_file(file_path: str) -> Tuple[List[Tuple[str, str]], List[Tuple[str, str]]]:
    """Parse a service file and return tuple of (request_fields, response_fields)"""
    request_fields = []
    response_fields = []
    current_section = request_fields
    
    with open(file_path, 'r') as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith('#'):
                continue
            
            # Check for section separator
            if line == '---':
                current_section = response_fields
                continue
            
            # Match field definition
            match = re.match(r'^([a-zA-Z0-9_/]+(?:\[\d*\])?)\s+([a-zA-Z][a-zA-Z0-9_]*)(.*?)$', line)
            if match:
                type_part = match.group(1)
                field_name = match.group(2)
                
                # Skip constant definitions
                if '=' in line:
                    continue
                
                # Process type and field name
                new_type = capitalize_type(type_part)
                current_section.append((new_type, field_name))
    
    return request_fields, response_fields

def compare_service_fields(ros1_fields: List[Tuple[str, str]], 
                         ros2_fields: List[Tuple[str, str]]) -> Optional[Dict[str, str]]:
    """Compare service fields and return field mapping if different"""
    if len(ros1_fields) != len(ros2_fields):
        return {ros1_field: ros2_field for (_, ros1_field), (_, ros2_field) in zip(ros1_fields, ros2_fields)}
    
    field_mapping = {}
    for (ros1_type, ros1_field), (ros2_type, ros2_field) in zip(ros1_fields, ros2_fields):
        # 检查字段名是否符合 snake_case 规则
        ros1_snake = camel_to_snake(ros1_field)
        ros2_snake = camel_to_snake(ros2_field)
        
        # 如果原始字段名不符合 snake_case 规则，添加到映射中
        if ros1_field != ros1_snake or ros2_field != ros2_snake:
            field_mapping[ros1_field] = ros2_field
    
    return field_mapping if field_mapping else None

def find_service_files(package_path: str) -> List[str]:
    """Find all .srv files in the package"""
    srv_files = []
    srv_dir = os.path.join(package_path, 'srv')
    if os.path.exists(srv_dir):
        for file in os.listdir(srv_dir):
            if file.endswith('.srv'):
                srv_files.append(os.path.join(srv_dir, file))
    return srv_files

def generate_mapping_rules(ros1_path: str, ros2_path: str) -> List[Dict]:
    """Generate mapping rules between ROS1 and ROS2 messages and services"""
    mapping_rules = []
    
    # Get message and service files from both packages
    ros1_msgs = find_message_files(ros1_path)
    ros2_msgs = find_message_files(ros2_path)
    ros1_srvs = find_service_files(ros1_path)
    ros2_srvs = find_service_files(ros2_path)
    
    # Create lookup for ROS2 messages and services (case-insensitive)
    ros2_msg_map = {get_message_name(msg): msg for msg in ros2_msgs}
    ros2_srv_map = {get_message_name(srv): srv for srv in ros2_srvs}
    
    # Process all ROS1 messages
    for ros1_msg in ros1_msgs:
        ros1_msg_name = os.path.basename(ros1_msg)[:-4]  # Remove .msg extension
        ros1_pkg_name = os.path.basename(os.path.dirname(os.path.dirname(ros1_msg)))
        
        # Find corresponding ROS2 message (case-insensitive)
        ros1_msg_key = get_message_name(ros1_msg)
        ros2_msg = ros2_msg_map.get(ros1_msg_key)
        
        if ros2_msg:
            ros2_pkg_name = os.path.basename(os.path.dirname(os.path.dirname(ros2_msg)))
            ros2_msg_name = os.path.basename(ros2_msg)[:-4]  # Remove .msg extension
            
            # Parse message contents
            ros1_fields = parse_message_file(ros1_msg)
            ros2_fields = parse_message_file(ros2_msg)
            
            # Compare messages
            field_mapping = compare_messages(ros1_fields, ros2_fields)
            
            # Create mapping rule
            rule = {
                'ros1_package_name': ros1_pkg_name,
                'ros1_message_name': ros1_msg_name,
                'ros2_package_name': ros2_pkg_name,
                'ros2_message_name': ros2_msg_name
            }
            
            if field_mapping:
                rule['fields_1_to_2'] = field_mapping
            
            mapping_rules.append(rule)
        else:
            # If no corresponding ROS2 message found, still create a rule
            rule = {
                'ros1_package_name': ros1_pkg_name,
                'ros1_message_name': ros1_msg_name,
                'ros2_package_name': ros1_pkg_name,  # Use same package name
                'ros2_message_name': ros1_msg_name   # Use same message name
            }
            mapping_rules.append(rule)
    
    # Process ROS2 messages that don't have corresponding ROS1 messages
    ros1_msg_keys = {get_message_name(msg) for msg in ros1_msgs}
    for ros2_msg in ros2_msgs:
        ros2_msg_key = get_message_name(ros2_msg)
        if ros2_msg_key not in ros1_msg_keys:
            ros2_msg_name = os.path.basename(ros2_msg)[:-4]  # Remove .msg extension
            ros2_pkg_name = os.path.basename(os.path.dirname(os.path.dirname(ros2_msg)))
            
            rule = {
                'ros1_package_name': ros2_pkg_name,  # Use same package name
                'ros1_message_name': ros2_msg_name,  # Use same message name
                'ros2_package_name': ros2_pkg_name,
                'ros2_message_name': ros2_msg_name
            }
            mapping_rules.append(rule)
    
    # Process all ROS1 services
    for ros1_srv in ros1_srvs:
        ros1_srv_name = os.path.basename(ros1_srv)[:-4]  # Remove .srv extension
        ros1_pkg_name = os.path.basename(os.path.dirname(os.path.dirname(ros1_srv)))
        
        # Find corresponding ROS2 service (case-insensitive)
        ros1_srv_key = get_message_name(ros1_srv)
        ros2_srv = ros2_srv_map.get(ros1_srv_key)
        
        if ros2_srv:
            ros2_pkg_name = os.path.basename(os.path.dirname(os.path.dirname(ros2_srv)))
            ros2_srv_name = os.path.basename(ros2_srv)[:-4]  # Remove .srv extension
            
            # Parse service contents
            ros1_request_fields, ros1_response_fields = parse_service_file(ros1_srv)
            ros2_request_fields, ros2_response_fields = parse_service_file(ros2_srv)
            
            # Compare service fields
            request_mapping = compare_service_fields(ros1_request_fields, ros2_request_fields)
            response_mapping = compare_service_fields(ros1_response_fields, ros2_response_fields)
            
            # Create mapping rule
            rule = {
                'ros1_package_name': ros1_pkg_name,
                'ros1_service_name': ros1_srv_name,
                'ros2_package_name': ros2_pkg_name,
                'ros2_service_name': ros2_srv_name
            }
            
            if request_mapping:
                rule['request_fields_1_to_2'] = request_mapping
            if response_mapping:
                rule['response_fields_1_to_2'] = response_mapping
            
            mapping_rules.append(rule)
        else:
            # If no corresponding ROS2 service found, still create a rule
            rule = {
                'ros1_package_name': ros1_pkg_name,
                'ros1_service_name': ros1_srv_name,
                'ros2_package_name': ros1_pkg_name,  # Use same package name
                'ros2_service_name': ros1_srv_name   # Use same service name
            }
            mapping_rules.append(rule)
    
    # Process ROS2 services that don't have corresponding ROS1 services
    ros1_srv_keys = {get_message_name(srv) for srv in ros1_srvs}
    for ros2_srv in ros2_srvs:
        ros2_srv_key = get_message_name(ros2_srv)
        if ros2_srv_key not in ros1_srv_keys:
            ros2_srv_name = os.path.basename(ros2_srv)[:-4]  # Remove .srv extension
            ros2_pkg_name = os.path.basename(os.path.dirname(os.path.dirname(ros2_srv)))
            
            rule = {
                'ros1_package_name': ros2_pkg_name,  # Use same package name
                'ros1_service_name': ros2_srv_name,  # Use same service name
                'ros2_package_name': ros2_pkg_name,
                'ros2_service_name': ros2_srv_name
            }
            mapping_rules.append(rule)
    
    return mapping_rules

def main():
    parser = argparse.ArgumentParser(description='Generate mapping rules between ROS1 and ROS2 messages')
    parser.add_argument('--ros1_path', help='Path to ROS1 package')
    parser.add_argument('--ros2_path', help='Path to ROS2 package')
    parser.add_argument('--output', '-o', default='mapping_rules.yaml',
                      help='Output YAML file (default: mapping_rules.yaml)')
    
    args = parser.parse_args()
    
    # Generate mapping rules
    mapping_rules = generate_mapping_rules(args.ros1_path, args.ros2_path)
    
    # Write to YAML file with spacing between rules
    with open(args.output, 'w') as f:
        for rule in mapping_rules:
            yaml.dump([rule], f, default_flow_style=False, sort_keys=False)
            f.write('\n')  # Add extra newline between rules
    
    print(f"Generated mapping rules written to {args.output}")

if __name__ == "__main__":
    main()

```

使用方法：

```bash
python generate_mapping_rules.py --ros1_path <your_ros1_ws>/src/ros1_msgs --ros2_path <your_ros2_ws>/src/ros2_msgs --output mapping_rules.yaml
```

## 编译 msg 和 bridge

官方提供编译操作比较繁琐，为了方便使用，我写了个 `bash` 脚本用于编译 `msg` 和 `bridge` 。


```bash
#!/bin/bash

current_dir=$(pwd)

# setup ros environment
ROS1_SETUP="/opt/ros/noetic/setup.bash"
ROS2_SETUP="/opt/ros/foxy/setup.bash"

if [ ! -f "$ROS1_SETUP" ]; then
    echo "Error: ROS1 setup file not found at $ROS1_SETUP"
    exit 1
fi

if [ ! -f "$ROS2_SETUP" ]; then
    echo "Error: ROS2 setup file not found at $ROS2_SETUP"
    exit 1
fi

echo "Building ROS1 packages..."
source "$ROS1_SETUP"
cd "$current_dir/ros1" || exit
rm -rf build_isolated install_isolated log_isolated || true
catkin_make_isolated --install


echo "Building ROS2 packages..."
source "$ROS2_SETUP"
cd "$current_dir/ros2" || exit
rm -rf build install log || true
colcon build


# check workspace install path
ROS1_INSTALL="$current_dir/ros1/install_isolated/setup.bash"
ROS2_INSTALL="$current_dir/ros2/install/local_setup.bash"
BRIDGE_INSTALL="$current_dir/bridge_ws/install/setup.bash"

if [ ! -f "$ROS1_INSTALL" ]; then
    echo "Error: ROS1 install setup file not found at $ROS1_INSTALL"
    exit 1
fi

if [ ! -f "$ROS2_INSTALL" ]; then
    echo "Error: ROS2 install setup file not found at $ROS2_INSTALL"
    exit 1
fi

# bridge
echo "Building bridge..."
cd "$current_dir/bridge_ws" || exit
rm -rf build install log || true
source "$ROS1_SETUP"
source "$ROS2_SETUP"
source "$ROS1_INSTALL"
source "$ROS2_INSTALL"

colcon build --symlink-install --packages-select ros1_bridge --cmake-force-configure
```

添加执行权限后，执行 `./build_msg_and_bridge.sh` 即可。

NOTE:

- 确认您的项目结构与预期匹配，期望如下: https://github.com/ros2/ros1_bridge/blob/master/doc/index.rst#example-workspace-setup
- 确认您的 `ros1` 和 `ros2` 的版本号，并修改 `build_msg_and_bridge.sh` 中的 `ROS1_SETUP` 和 `ROS2_SETUP` 的版本号。

## 遇到的问题

### 桥接 `service` 时，无法通过编译

上面提到如果需要桥接 `customized` 的 `interface` ，则需要手动生成 `rule` 映射文件。

按照官方文档操作后，发现 `ros1_bridge` 编译失败。

然而这不是我们预期的结果，经过一轮抽丝剥茧，发现是接口工程模版文件的问题。

关键问题：

```cpp
auto @(field["ros1"]["name"])1_it = req1.@(field["ros1"]["name"]).begin();
auto @(field["ros2"]["name"])2_it = req2.@(field["ros2"]["name"]).begin();
@[        if field["basic"]]@
  @(field["ros2"]["name"])@(to) = @(field["ros1"]["name"])@(frm);
@[        else]@
  Factory<@(field["ros1"]["cpptype"]),@(field["ros2"]["cpptype"])>::convert_@(frm)_to_@(to)(@
@(field["ros2"]["name"])@(frm), @(field["ros1"]["name"])@(to));
```

source code: https://github.com/ros2/ros1_bridge/blob/6c5ad167139ae64a42c8a202614f7c84004081ee/resource/interface_factories.cpp.em#L324

这里判断是假如不是 `basic` 类型，则调用 `Factory` 的 `convert_` 函数。

问题在与参数传递存在问题, 上述的逻辑是假设了 `ros1` 和 `ros2` 的 `field` 是相同的(相同的又何必规则映射...)

例子：

```yaml
- ros1_package_name: my_msgs
  ros1_service_name: my_srv
  ros2_package_name: my_msgs
  ros2_service_name: MySrv
  request_fields_1_to_2:
    Req: req
```

编译出生成的接口工程代码：

```cpp
template <>
void ServiceFactory<
  my_msgs::MySrv,
  my_msgs::srv::MySrv
>::translate_1_to_2(
  const my_msgs::MySrv::Request& req1,
  my_msgs::srv::MySrv::Request& req2
) {
  auto & Req1 = req1.req;
  auto & req2 = req2.req;
  Factory<my_msgs::MyMsg,my_msgs::msg::MyMsg>::convert_1_to_2(
    req1, Req2
  );
}
```

这个时候会出现错误提示：`req1` 和 `Req2 was not declared in this scope`


### 解决问题

```cpp
  Factory<@(field["ros1"]["cpptype"]),@(field["ros2"]["cpptype"])>::convert_@(frm)_to_@(to)(
@[          if frm == "1"]@
    @(field["ros1"]["name"])1, @(field["ros2"]["name"])2
@[          else]@
    @(field["ros2"]["name"])2, @(field["ros1"]["name"])1
@[          end if]@
  );
```

改成这样就好啦，根据到底是 `convert_1_to_2` 还是 `convert_2_to_1` 来决定参数的顺序。

已经提出 Issue 和 PR 给官方仓库，等待合并。

Issue: https://github.com/ros2/ros1_bridge/issues/455
PR: https://github.com/ros2/ros1_bridge/pull/456
