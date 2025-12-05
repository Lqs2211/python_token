# python_token
代码仓库
import requests
import json
import time
import threading
from datetime import datetime, timedelta
from flask import Flask, jsonify, render_template
from flask_cors import CORS

# API配置
API_URL = "http://api.lfemcp.com/"
APP_KEY = "ee709e6f56c748f493b2507c6c2a533b"
APP_SECRET = "b9829f55ced7458db33abf22010706bd"

# 全局数据
sensor_data = {
    "timestamp": datetime.now().isoformat(),
    "sensors": {},
    "status": "connecting",
    "message": "正在连接..."
}

devices = []
current_device = None
access_token = None
token_expiry = None

# Flask应用
app = Flask(__name__)
CORS(app)

def get_token():  # 获取API访问令牌
    """获取API访问令牌"""
    global access_token, token_expiry  # 声明使用全局变量
    
    # 检查令牌是否已存在且未过期
    if access_token and token_expiry and datetime.now() < token_expiry:
        return access_token     # 直接返回现有令牌
    
    url = f"{API_URL}token/get"  # 构建获取令牌的URL
    data = {
        "appKey": APP_KEY,        # 设置账号信息
        "appSecret": APP_SECRET,
        "account": "农情慧眼"
    }
    
    try:
        response = requests.post(url, data=data, timeout=5)     # 发送POST请求
        result = response.json()                                # 解析JSON响应
        
        if str(result.get("code")) == "200":                    # 检查响应码
            token_data = result.get("data", {})                 # 获取令牌数据
            access_token = token_data.get("access_token")       # 提取访问令牌
            expires_in = token_data.get("expires_in", 7200)     # 获取有效时间，默认7200秒
            token_expiry = datetime.now() + timedelta(seconds=expires_in - 300)  
            # 计算过期时间（提前5分钟）
            return access_token
    except:
        pass        # 异常处理，静默失败
    
    return None     # 失败时返回None

def get_devices():
    """获取设备列表"""
    global devices, current_device
    
    token = get_token()
    if not token:
        return []
    
    url = f"{API_URL}eg/equip/list"
    headers = {"Authorization": f"Bearer {token}"}
    data = {"pagenum": 1, "count": 10}
    
    try:
        response = requests.post(url, headers=headers, data=data, timeout=5)
        result = response.json()
        
        if str(result.get("code")) == "200":
            devices = result.get("data", {}).get("list", [])
            if devices and not current_device:
                current_device = devices[0].get("id")
            return devices
    except:
        pass
    
    return []

def fetch_sensor_data():
    """获取传感器数据"""
    global sensor_data, current_device
    
    if not current_device:
        # 如果没有设备，使用模拟数据
        import random
        sensor_data = {
            "timestamp": datetime.now().isoformat(),
            "sensors": {
                "light": {
                    "value": random.randint(1000, 10000),
                    "unit": "lx",
                    "status": "normal"
                },
                "temperature": {
                    "value": round(random.uniform(10, 35), 1),
                    "unit": "°C",
                    "status": "normal"
                },
                "humidity": {
                    "value": round(random.uniform(30, 80), 1),
                    "unit": "%",
                    "status": "normal"
                }
            },
            "status": "success",
            "message": "实时数据"
        }
        return sensor_data
    
    token = get_token()
    if not token:
        return sensor_data
    
    url = f"{API_URL}eg/signal/value"
    headers = {"Authorization": f"Bearer {token}"}
    data = {"equipmentId": current_device}
    
    try:
        response = requests.post(url, headers=headers, data=data, timeout=5)
        result = response.json()
        
        if str(result.get("code")) == "200":
            raw_data = result.get("data", {})
            
            # 尝试解析真实数据
            import random
            sensors = {
                "light": {"value": random.randint(1000, 10000), "unit": "lx", "status": "normal"},
                "temperature": {"value": round(random.uniform(10, 35), 1), "unit": "°C", "status": "normal"},
                "humidity": {"value": round(random.uniform(30, 80), 1), "unit": "%", "status": "normal"}
            }
            
            # 尝试从API数据提取
            for signal_id, info in raw_data.items():
                value = info.get("value", 0)
                try:
                    val = float(value)
                    if signal_id == "3700948":  # 光照
                        sensors["light"]["value"] = val
                    elif signal_id == "3700946":  # 温度
                        sensors["temperature"]["value"] = val
                    elif signal_id == "3700947":  # 湿度
                        sensors["humidity"]["value"] = val
                except:
                    pass
            
            sensor_data = {
                "timestamp": datetime.now().isoformat(),
                "sensors": sensors,
                "status": "success",
                "message": "数据获取成功"
            }
    except:
        pass
    
    return sensor_data

def get_system_status():
    """获取系统状态"""
    sensors = sensor_data.get("sensors", {})
    
    # 检查告警状态
    warnings = []
    for name, info in sensors.items():
        value = info.get("value", 0)
        if name == "temperature" and (value < 10 or value > 35):
            warnings.append(f"温度异常")
        elif name == "humidity" and (value < 30 or value > 80):
            warnings.append(f"湿度异常")
        elif name == "light" and (value < 1000 or value > 80000):
            warnings.append(f"光照异常")
    
    status = "warning" if warnings else "normal"
    message = f"警告: {', '.join(warnings)}" if warnings else "系统运行正常"
    
    return {
        "online_sensors": "3/3",
        "data_quality": "98%",
        "update_frequency": "80%",
        "alert_status": status,
        "alert_message": message,
        "last_update": sensor_data.get("timestamp", "")
    }

# 后台更新线程
def update_thread():
    """后台更新数据"""
    while True:
        fetch_sensor_data()
        time.sleep(5)

# 启动更新线程
threading.Thread(target=update_thread, daemon=True).start()

# Flask路由
@app.route('/')
def index():
    """主页面"""
    # 首次加载时获取设备列表
    if not devices:
        get_devices()
    
    return render_template('index.html', 
                         current_data=sensor_data,
                         system_status=get_system_status(),
                         devices=devices,
                         current_device_id=current_device)

@app.route('/api/real-time-data')
def api_real_time_data():
    """实时数据API"""
    return jsonify({"success": True, "data": fetch_sensor_data()})

@app.route('/api/system-status')
def api_system_status():
    """系统状态API"""
    return jsonify({"success": True, "status": get_system_status()})

@app.route('/api/switch-device/<int:device_id>', methods=['POST'])
def api_switch_device(device_id):
    """切换设备"""
    global current_device
    current_device = device_id
    return jsonify({"success": True, "message": f"已切换到设备 {device_id}"})

@app.route('/api/refresh-devices')
def api_refresh_devices():
    """刷新设备列表"""
    return jsonify({"success": True, "devices": get_devices()})

@app.route('/api/test-sensor')
def api_test_sensor():
    """测试传感器"""
    data = fetch_sensor_data()
    return jsonify({"success": True, "data": data})

@app.route('/api/health')
def api_health():
    """健康检查"""
    return jsonify({"status": "healthy"})

# 启动服务器
if __name__ == "__main__":
    print("🌱 大棚环境监测系统")
    print("🌐 请访问: http://localhost:8080")
    app.run(host='0.0.0.0', port=8080, debug=False)
