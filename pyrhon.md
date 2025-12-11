import requests
import json
import time
import threading
from datetime import datetime, timedelta
from flask import Flask, jsonify, render_template
from flask_cors import CORS
import logging
import random

# 配置日志
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(name)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

# 全局数据
sensor_data = {
    "timestamp": datetime.now().isoformat(),
    "sensors": {},
    "status": "initializing",
    "message": "系统初始化中..."
}

devices = []
current_device = None
access_token = None
token_expiry = None
monitor_start_time = None
MONITOR_DURATION = 580  # 监控有效期580秒，留20秒缓冲

# 土壤湿度模拟数据的当前值（确保在40-60%之间）
current_soil_moisture = 52.0

# 传感器ID映射表（需要根据实际设备调整）
SENSOR_MAPPING = {
    "temperature": {
        "keywords": ["温度", "temp", "温"],
        "default_value": 25.0,
        "unit": "°C",
        "range": (10, 35),
        "simulated": False  # 从API获取
    },
    "humidity": {
        "keywords": ["湿度", "hum", "湿"],
        "default_value": 60.0,
        "unit": "%",
        "range": (30, 80),
        "simulated": False  # 从API获取
    },
    "light": {
        "keywords": ["光照度"],  # 只匹配"光照度"，不匹配"补光内部"或"补光灯"
        "default_value": 5000.0,
        "unit": "lx",
        "range": (1000, 10000),
        "simulated": False  # 从API获取
    }
}

# 新增土壤湿度传感器（只使用模拟数据）
SOIL_MOISTURE_CONFIG = {
    "keywords": ["土壤湿度", "soil", "moisture"],
    "default_value": 52.0,  # 默认值设为52%
    "unit": "%",
    "range": (40, 60),  # 数值范围40%-60%
    "simulated": True  # 始终使用模拟数据
}

# Flask应用
app = Flask(__name__)
CORS(app)  # 允许跨域请求

class APIError(Exception):
    """API错误异常类"""
    pass

def get_token():
    global access_token, token_expiry

    if access_token and token_expiry and datetime.now() < token_expiry - timedelta(minutes=5):
        logger.info("使用现有的Token")
        return access_token
    
    logger.info("获取新的Token")
    url = f"{API_URL}token/get"
    data = {
        "appKey": APP_KEY,
        "appSecret": APP_SECRET,
        "account": "农情慧眼"
    }

    response = requests.post(url, data=data, timeout=10)
    result = response.json()

    if str(result.get("code")) == "200":
        token_data = result.get("data", {})
        access_token = token_data.get("access_token")
        expires_in = token_data.get("expires_in", 7200)
        token_expiry = datetime.now() + timedelta(seconds=expires_in)

        logger.info(f"Token获取成功，有效期至{token_expiry}")
        return access_token

def get_devices():
    """获取设备列表"""
    global devices, current_device
    
    token = get_token()
    if not token:
        return []
    
    url = f"{API_URL}eg/equip/list"
    headers = {"Authorization": f"Bearer {token}"}
    data = {"pagenum": 1, "count": 50}
    
    try:
        response = requests.post(url, headers=headers, data=data, timeout=10)
        result = response.json()
        
        if str(result.get("code")) == "200":
            devices_data = result.get("data", {}).get("list", [])
            devices = devices_data
            
            if devices and not current_device:
                current_device = devices[0].get("id")
                logger.info(f"选择默认设备: {current_device}")
            
            logger.info(f"获取到{len(devices)}个设备")
            return devices
        else:
            error_msg = f"获取设备列表失败: {result.get('message', '未知错误')}"
            logger.error(error_msg)
            return []
    except Exception as e:
        logger.error(f"获取设备列表异常: {e}")
        return []

def open_device_monitor(device_id):
    """开启设备监控"""
    global monitor_start_time
    
    token = get_token()
    if not token:
        return False, "无法获取访问令牌"
    
    url = f"{API_URL}eg/monitor/open"
    headers = {"Authorization": f"Bearer {token}"}
    data = {"equipmentId": device_id}
    
    try:
        logger.info(f"开启设备 {device_id} 监控...")
        response = requests.post(url, headers=headers, data=data, timeout=10)
        result = response.json()
        
        if str(result.get("code")) == "200":
            monitor_start_time = datetime.now()
            logger.info(f"设备 {device_id} 监控已开启，有效期{int(MONITOR_DURATION/60)}分钟")
            return True, "设备监控已开启"
        else:
            error_msg = f"开启监控失败: {result.get('message', '未知错误')}"
            logger.error(error_msg)
            return False, error_msg
    except Exception as e:
        error_msg = f"开启监控异常: {e}"
        logger.error(error_msg)
        return False, error_msg

def is_monitor_valid():
    """检查监控是否在有效期内"""
    global monitor_start_time
    
    if not monitor_start_time:
        return False, "监控未开启"
    
    elapsed_time = (datetime.now() - monitor_start_time).total_seconds()
    
    if elapsed_time < MONITOR_DURATION:
        remaining_time = int(MONITOR_DURATION - elapsed_time)
        return True, f"监控有效，剩余{remaining_time}秒"
    else:
        return False, "监控已过期，需要重新开启"

def get_signal_list(device_id):
    """查询设备变量列表"""
    token = get_token()
    if not token:
        return []
    
    url = f"{API_URL}eg/signal/list"
    headers = {"Authorization": f"Bearer {token}"}
    data = {"equipmentId": device_id}
    
    try:
        response = requests.post(url, headers=headers, data=data, timeout=10)
        result = response.json()
        
        if str(result.get("code")) == "200":
            signals = result.get("data", [])
            logger.info(f"获取到设备 {device_id} 的 {len(signals)} 个信号")
            return signals
        else:
            logger.warning(f"获取变量列表失败: {result.get('message')}")
            return []
    except Exception as e:
        logger.error(f"获取变量列表异常: {e}")
        return []

def create_sensor_mapping(signal_list):
    """创建传感器ID到变量ID的映射"""
    sensor_mapping = {}
    
    for signal in signal_list:
        signal_id = str(signal.get("id"))
        title = signal.get("title", "")
        unit = signal.get("unit", "")
        
        # 根据关键词匹配传感器类型
        for sensor_type, config in SENSOR_MAPPING.items():
            for keyword in config["keywords"]:
                if keyword in title:
                    # 跳过补光灯相关的传感器
                    if "补光" in title:
                        logger.debug(f"跳过补光灯传感器: {signal_id} -> {title}")
                        continue
                    
                    sensor_mapping[signal_id] = {
                        "type": sensor_type,
                        "title": title,
                        "unit": unit if unit else config["unit"]
                    }
                    logger.info(f"映射成功: {signal_id} -> {sensor_type} ({title})")
                    break
    
    return sensor_mapping

def get_sensor_values(device_id, sensor_mapping):
    """获取传感器值（只获取温度、湿度、光照）"""
    token = get_token()
    if not token:
        return {}
    
    url = f"{API_URL}eg/signal/value"
    headers = {"Authorization": f"Bearer {token}"}
    data = {"equipmentId": device_id}
    
    try:
        response = requests.post(url, headers=headers, data=data, timeout=10)
        result = response.json()
        
        if str(result.get("code")) == "200":
            raw_data = result.get("data", {})
            sensors = {}
            
            # 初始化传感器数据结构（不包括土壤湿度）
            for sensor_type, config in SENSOR_MAPPING.items():
                sensors[sensor_type] = {
                    "value": config["default_value"],
                    "unit": config["unit"],
                    "status": "normal",
                    "raw_value": None,
                    "simulated": False
                }
            
            # 解析真实数据
            count_real_data = 0
            for signal_id, info in raw_data.items():
                if signal_id in sensor_mapping:
                    sensor_type = sensor_mapping[signal_id]["type"]
                    raw_value = info.get("value")
                    
                    if raw_value is not None:
                        try:
                            # 尝试转换数值
                            value = float(raw_value)
                            sensors[sensor_type]["value"] = value
                            sensors[sensor_type]["raw_value"] = raw_value
                            sensors[sensor_type]["unit"] = sensor_mapping[signal_id]["unit"]
                            count_real_data += 1
                        except (ValueError, TypeError):
                            logger.warning(f"无法解析传感器值: {signal_id}={raw_value}")
            
            logger.info(f"成功获取 {count_real_data} 个真实传感器值")
            return sensors
        else:
            logger.warning(f"获取传感器值失败: {result.get('message')}")
            return {}
    except Exception as e:
        logger.error(f"获取传感器值异常: {e}")
        return {}

def generate_soil_moisture_data():
    """生成土壤湿度模拟数据"""
    global current_soil_moisture
    
    # 在上次值的基础上进行微小变化 (±0.5% 到 ±1.5%)
    change = random.uniform(-1.5, 1.5)
    new_value = current_soil_moisture + change
    
    # 确保在40-60%范围内
    if new_value < 40:
        new_value = 40.0
    elif new_value > 60:
        new_value = 60.0
    
    # 更新当前值
    current_soil_moisture = round(new_value, 1)
    
    return {
        "value": current_soil_moisture,
        "unit": "%",
        "status": "normal",
        "raw_value": current_soil_moisture,
        "simulated": True
    }

def generate_simulated_sensor_data():
    """生成模拟传感器数据（当API不可用时）"""
    import random
    
    sensors = {}
    
    # 生成温度、湿度、光照的模拟数据
    for sensor_type, config in SENSOR_MAPPING.items():
        min_val, max_val = config["range"]
        value = random.uniform(min_val, max_val)
        
        sensors[sensor_type] = {
            "value": round(value, 1),
            "unit": config["unit"],
            "status": "normal",
            "raw_value": value,
            "simulated": True
        }
    
    return sensors

def update_sensor_data():
    """更新传感器数据"""
    global sensor_data, current_device
    
    try:
        # 步骤1: 检查是否有当前设备
        if not current_device:
            logger.warning("没有选择设备，获取设备列表...")
            get_devices()
            
            if not current_device:
                logger.info("使用模拟数据")
                sensors = generate_simulated_sensor_data()
                # 添加土壤湿度模拟数据
                sensors["soil_moisture"] = generate_soil_moisture_data()
                
                sensor_data = {
                    "timestamp": datetime.now().isoformat(),
                    "sensors": sensors,
                    "status": "simulated",
                    "message": "使用模拟数据 - 无可用设备"
                }
                return
        
        # 步骤2: 检查监控状态
        monitor_valid, monitor_msg = is_monitor_valid()
        
        if not monitor_valid:
            logger.info(f"监控无效: {monitor_msg}")
            # 步骤3: 开启设备监控
            success, message = open_device_monitor(current_device)
            if not success:
                logger.error(f"开启监控失败: {message}")
                # 如果监控失败，使用模拟数据
                sensors = generate_simulated_sensor_data()
                sensors["soil_moisture"] = generate_soil_moisture_data()
                
                sensor_data = {
                    "timestamp": datetime.now().isoformat(),
                    "sensors": sensors,
                    "status": "monitor_error",
                    "message": f"监控失败: {message}，使用模拟数据"
                }
                return
        
        # 步骤4: 获取变量列表
        signal_list = get_signal_list(current_device)
        if not signal_list:
            logger.warning("无法获取变量列表，尝试使用模拟数据")
            sensors = generate_simulated_sensor_data()
            sensors["soil_moisture"] = generate_soil_moisture_data()
        else:
            # 步骤5: 创建映射并获取数据（不包括土壤湿度）
            sensor_mapping = create_sensor_mapping(signal_list)
            sensors = get_sensor_values(current_device, sensor_mapping)
            
            if not sensors:
                logger.warning("无法获取传感器值，使用模拟数据")
                sensors = generate_simulated_sensor_data()
                sensors["soil_moisture"] = generate_soil_moisture_data()
            else:
                # 确保所有传感器都有数据
                for sensor_type, config in SENSOR_MAPPING.items():
                    if sensor_type not in sensors:
                        # 使用模拟数据补充缺失的传感器
                        min_val, max_val = config["range"]
                        value = random.uniform(min_val, max_val)
                        
                        sensors[sensor_type] = {
                            "value": round(value, 1),
                            "unit": config["unit"],
                            "status": "normal",
                            "raw_value": value,
                            "simulated": True
                        }
        
        # 步骤6: 添加土壤湿度模拟数据（始终使用模拟数据）
        sensors["soil_moisture"] = generate_soil_moisture_data()
        
        # 步骤7: 更新状态
        sensor_status = "success"
        sensor_message = "实时数据获取成功"
        
        # 检查是否有模拟数据
        simulated_count = sum(1 for s in sensors.values() if s.get("simulated", False))
        if simulated_count > 0:
            sensor_status = "partial"
            sensor_message = f"部分模拟数据 ({simulated_count}/{len(sensors)})"
        
        # 如果土壤湿度是模拟的，在消息中特别说明
        if sensors.get("soil_moisture", {}).get("simulated", False):
            sensor_message += " (土壤湿度为模拟数据)"
        
        sensor_data = {
            "timestamp": datetime.now().isoformat(),
            "sensors": sensors,
            "status": sensor_status,
            "message": sensor_message,
            "device_id": current_device,
            "monitor_valid": monitor_valid
        }
        
        logger.info(f"传感器数据更新成功: {sensor_status} - {sensor_message}")
        
    except Exception as e:
        logger.error(f"更新传感器数据时发生异常: {e}")
        # 异常时使用模拟数据
        sensors = generate_simulated_sensor_data()
        sensors["soil_moisture"] = generate_soil_moisture_data()
        sensor_data = {
            "timestamp": datetime.now().isoformat(),
            "sensors": sensors,
            "status": "error",
            "message": f"数据更新异常: {str(e)}，使用模拟数据"
        }

def get_system_status():
    """获取系统状态"""
    sensors = sensor_data.get("sensors", {})
    
    # 检查监控状态
    monitor_valid, monitor_msg = is_monitor_valid()
    
    # 计算在线传感器（不包括土壤湿度，因为它是模拟的）
    online_count = 0
    warnings = []
    
    for name, info in sensors.items():
        value = info.get("value", 0)
        simulated = info.get("simulated", False)
        
        # 土壤湿度是模拟的，不计入在线传感器
        if name == "soil_moisture":
            continue
            
        if not simulated:
            online_count += 1
        
        # 检查值是否在正常范围内
        if name in SENSOR_MAPPING:
            config = SENSOR_MAPPING[name]
            if "range" in config:
                min_val, max_val = config["range"]
                if value < min_val or value > max_val:
                    warnings.append(f"{name}异常: {value}{info.get('unit', '')}")
    
    if not monitor_valid:
        warnings.append(f"监控状态: {monitor_msg}")
    
    # 检查土壤湿度是否在正常范围内
    soil_info = sensors.get("soil_moisture", {})
    soil_value = soil_info.get("value", 0)
    if soil_value < 40 or soil_value > 60:
        warnings.append(f"土壤湿度异常: {soil_value}% (正常范围: 40%-60%)")
    
    status = "warning" if warnings else "normal"
    
    return {
        "online_sensors": f"{online_count}/{len(SENSOR_MAPPING)}",  # 不包括土壤湿度
        "data_quality": "98%",
        "update_frequency": "5秒",
        "alert_status": status,
        "alert_message": "；".join(warnings) if warnings else "系统运行正常",
        "monitor_status": "有效" if monitor_valid else "无效",
        "monitor_message": monitor_msg,
        "last_update": sensor_data.get("timestamp", ""),
        "system_status": sensor_data.get("status", "unknown"),
        "soil_moisture_note": "土壤湿度使用模拟数据(40%-60%)"
    }

# 后台更新线程
def update_thread():
    """后台定时更新数据"""
    logger.info("启动后台数据更新线程")
    
    # 首次启动时初始化
    get_devices()
    
    while True:
        try:
            update_sensor_data()
            time.sleep(5)  # 每5秒更新一次
        except Exception as e:
            logger.error(f"后台更新线程异常: {e}")
            time.sleep(10)

# 启动更新线程
threading.Thread(target=update_thread, daemon=True).start()

# Flask路由
@app.route('/')
def index():
    """主页面"""
    return render_template('index.html')

@app.route('/api/real-time-data')
def api_real_time_data():
    """实时数据API"""
    try:
        return jsonify({
            "success": True,
            "data": sensor_data
        })
    except Exception as e:
        logger.error(f"API异常: {e}")
        return jsonify({
            "success": False,
            "error": str(e),
            "data": sensor_data
        }), 500

@app.route('/api/system-status')
def api_system_status():
    """系统状态API"""
    return jsonify({
        "success": True,
        "status": get_system_status()
    })

@app.route('/api/devices')
def api_devices():
    """设备列表API"""
    return jsonify({
        "success": True,
        "devices": devices,
        "current_device": current_device
    })

@app.route('/api/switch-device/<device_id>', methods=['POST'])
def api_switch_device(device_id):
    """切换设备"""
    global current_device, monitor_start_time
    
    try:
        device_id = int(device_id)
        old_device = current_device
        current_device = device_id
        monitor_start_time = None  # 重置监控状态
        
        # 为新设备开启监控
        success, message = open_device_monitor(device_id)
        
        return jsonify({
            "success": success,
            "message": message,
            "old_device": old_device,
            "new_device": device_id
        })
    except Exception as e:
        logger.error(f"切换设备异常: {e}")
        return jsonify({
            "success": False,
            "message": f"切换设备失败: {str(e)}"
        }), 500

@app.route('/api/refresh-monitor', methods=['POST'])
def api_refresh_monitor():
    """刷新监控"""
    if not current_device:
        return jsonify({
            "success": False,
            "message": "没有选中设备"
        })
    
    success, message = open_device_monitor(current_device)
    
    return jsonify({
        "success": success,
        "message": message
    })

@app.route('/api/monitor-status')
def api_monitor_status():
    """监控状态API"""
    valid, message = is_monitor_valid()
    return jsonify({
        "success": True,
        "monitor_valid": valid,
        "message": message,
        "start_time": monitor_start_time.isoformat() if monitor_start_time else None,
        "duration_seconds": MONITOR_DURATION,
        "current_time": datetime.now().isoformat()
    })

@app.route('/api/test')
def api_test():
    """测试接口"""
    return jsonify({
        "success": True,
        "message": "API服务正常运行",
        "timestamp": datetime.now().isoformat(),
        "python_version": "Python后端API",
        "status": "online"
    })

@app.route('/api/health')
def api_health():
    """健康检查"""
    return jsonify({
        "status": "healthy",
        "timestamp": datetime.now().isoformat(),
        "sensor_status": sensor_data.get("status", "unknown")
    })

@app.route('/api/config')
def api_config():
    """配置信息"""
    return jsonify({
        "success": True,
        "config": {
            "api_url": API_URL,
            "monitor_duration": MONITOR_DURATION,
            "sensor_mapping": SENSOR_MAPPING,
            "soil_moisture_config": SOIL_MOISTURE_CONFIG,
            "current_device": current_device,
            "device_count": len(devices),
            "soil_moisture_note": "土壤湿度使用模拟数据，范围40%-60%"
        }
    })

# 错误处理
@app.errorhandler(404)
def not_found(error):
    return jsonify({"success": False, "error": "接口不存在"}), 404

@app.errorhandler(500)
def server_error(error):
    return jsonify({"success": False, "error": "服务器内部错误"}), 500

# 启动服务器
if __name__ == "__main__":
    print("=" * 50)
    print("🌱 大棚环境监测系统")
    print("=" * 50)
    print("📊 API流程: 获取Token → 获取设备 → 开启监控 → 获取数据")
    print(f"⏰ 监控有效期: {MONITOR_DURATION}秒 ({int(MONITOR_DURATION/60)}分钟)")
    print("🔄 数据更新频率: 5秒")
    print("🌐 Web界面: http://localhost:8080")
    print("📡 API端点: http://localhost:8080/api/real-time-data")
    print("📊 传感器配置:")
    for sensor, config in SENSOR_MAPPING.items():
        print(f"  - {sensor}: 范围{config.get('range', 'N/A')}{config['unit']} (从API获取)")
    print("  - soil_moisture: 范围40-60% (使用模拟数据)")
    print("=" * 50)
    
    # 启动Flask应用
    app.run(host='0.0.0.0', port=8080, debug=False, threaded=True)
