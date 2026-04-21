---
title: "Python 接口自动化测试从 0 到 1"
description: "用 Python + requests + pytest 搭建接口自动化测试框架，零基础也能上手。附完整代码。"
date: 2026-03-25
tags:
  - Python
  - 测试
  - 接口测试
---
> 接口测试是测试工程师的基本功，但很多人还在用 Postman 手工点。这篇文章教你用 Python 搭一套真正的自动化测试框架。

* * *

## 一、为什么用 Python 做接口测试？ 

方案 | 优点 | 缺点  
---|---|---  
Postman | 简单直观 | 不方便 CI 集成，团队协作弱  
JMeter | 性能测试强 | 界面复杂，脚本难维护  
**Python** | **灵活、可读性好、CI 友好、免费** | 需要写代码  
Java+RestAssured | 企业级 | 太重了  
  
**Python 的优势：代码即文档，Git 管理方便，Jenkins 集成简单，学习成本低。**

* * *

## 二、环境准备 
    
    
    # 安装依赖
    pip install requests pytest pytest-html allure-pytest
    
    # 项目结构
    api_test/
    ├── config/
    │   └── settings.py       # 配置文件（URL、账号等）
    ├── common/
    │   ├── base_api.py       # 封装的请求方法
    │   └── assertions.py     # 自定义断言
    ├── testcases/
    │   ├── test_login.py     # 登录测试
    │   ├── test_user.py      # 用户管理测试
    │   └── test_order.py     # 订单测试
    ├── data/
    │   └── test_data.yaml    # 测试数据
    ├── conftest.py           # pytest fixtures
    ├── pytest.ini            # pytest 配置
    └── requirements.txt
    

* * *

## 三、基础封装 

### config/settings.py 
    
    
    BASE_URL = "https://api.example.com"
    
    # 测试账号
    USERNAME = "test_user"
    PASSWORD = "test_password"
    
    # 超时设置
    TIMEOUT = 10
    

### common/base_api.py 
    
    
    import requests
    import logging
    
    logger = logging.getLogger(__name__)
    
    
    class BaseAPI:
        """封装 requests，统一处理请求和响应"""
        
        def __init__(self, base_url, token=None):
            self.base_url = base_url
            self.session = requests.Session()
            self.session.headers.update({
                "Content-Type": "application/json",
                "Accept": "application/json",
            })
            if token:
                self.session.headers["Authorization"] = f"Bearer {token}"
        
        def request(self, method, path, **kwargs):
            """统一请求方法"""
            url = f"{self.base_url}{path}"
            kwargs.setdefault("timeout", 10)
            
            logger.info(f"请求: {method} {url}")
            logger.debug(f"参数: {kwargs}")
            
            try:
                response = self.session.request(method, url, **kwargs)
                logger.info(f"响应: {response.status_code}")
                logger.debug(f"响应体: {response.text[:500]}")
                return response
            except requests.RequestException as e:
                logger.error(f"请求失败: {e}")
                raise
        
        def get(self, path, params=None):
            return self.request("GET", path, params=params)
        
        def post(self, path, json=None, data=None):
            return self.request("POST", path, json=json, data=data)
        
        def put(self, path, json=None):
            return self.request("PUT", path, json=json)
        
        def delete(self, path):
            return self.request("DELETE", path)
    

### conftest.py 
    
    
    import pytest
    from common.base_api import BaseAPI
    from config.settings import BASE_URL
    
    
    @pytest.fixture(scope="session")
    def api():
        """全局 API 实例"""
        return BaseAPI(BASE_URL)
    
    
    @pytest.fixture(scope="session")
    def auth_api(api):
        """带登录态的 API 实例"""
        response = api.post("/auth/login", json={
            "username": "test_user",
            "password": "test_password"
        })
        token = response.json()["data"]["token"]
        return BaseAPI(BASE_URL, token=token)
    

* * *

## 四、编写测试用例 

### testcases/test_login.py 
    
    
    import pytest
    
    
    class TestLogin:
        """登录接口测试"""
        
        def test_login_success(self, api):
            """正常登录"""
            resp = api.post("/auth/login", json={
                "username": "test_user",
                "password": "test_password"
            })
            
            assert resp.status_code == 200
            data = resp.json()
            assert data["code"] == 0
            assert "token" in data["data"]
            assert len(data["data"]["token"]) > 0
        
        def test_login_wrong_password(self, api):
            """密码错误"""
            resp = api.post("/auth/login", json={
                "username": "test_user",
                "password": "wrong_password"
            })
            
            assert resp.status_code == 200
            data = resp.json()
            assert data["code"] != 0
        
        def test_login_empty_username(self, api):
            """用户名为空"""
            resp = api.post("/auth/login", json={
                "username": "",
                "password": "test_password"
            })
            
            assert resp.status_code == 400  # 或 422，取决于接口设计
        
        @pytest.mark.parametrize("username, password, expected_code", [
            ("test_user", "test_password", 0),
            ("test_user", "wrong", 1001),
            ("not_exist", "test_password", 1002),
        ])
        def test_login_scenarios(self, api, username, password, expected_code):
            """参数化测试：多种登录场景"""
            resp = api.post("/auth/login", json={
                "username": username,
                "password": password
            })
            
            assert resp.json()["code"] == expected_code
    

### testcases/test_user.py 
    
    
    import pytest
    
    
    class TestUser:
        """用户管理接口测试"""
        
        def test_get_user_info(self, auth_api):
            """获取当前用户信息"""
            resp = auth_api.get("/user/info")
            
            assert resp.status_code == 200
            data = resp.json()
            assert data["code"] == 0
            assert data["data"]["username"] == "test_user"
        
        def test_update_nickname(self, auth_api):
            """修改昵称"""
            resp = auth_api.put("/user/info", json={
                "nickname": f"test_user_{int(time.time())}"
            })
            
            assert resp.status_code == 200
            assert resp.json()["code"] == 0
        
        def test_get_user_list(self, auth_api):
            """获取用户列表（需要管理员权限）"""
            resp = auth_api.get("/admin/users", params={"page": 1, "size": 10})
            
            assert resp.status_code == 200
            data = resp.json()
            assert "list" in data["data"]
            assert len(data["data"]["list"]) <= 10
    

* * *

## 五、数据驱动测试 

### data/test_data.yaml 
    
    
    login_cases:
      - name: 正常登录
        username: test_user
        password: test_password
        expected_code: 0
      - name: 密码错误
        username: test_user
        password: wrong123
        expected_code: 1001
      - name: 用户不存在
        username: ghost_user
        password: any123
        expected_code: 1002
    

### 使用 YAML 数据 
    
    
    import pytest
    import yaml
    
    
    def load_test_data():
        with open("data/test_data.yaml", "r", encoding="utf-8") as f:
            return yaml.safe_load(f)["login_cases"]
    
    
    @pytest.mark.parametrize("case", load_test_data())
    def test_login_data_driven(api, case):
        resp = api.post("/auth/login", json={
            "username": case["username"],
            "password": case["password"]
        })
        assert resp.json()["code"] == case["expected_code"]
    

* * *

## 六、运行测试 

### 基本命令 
    
    
    # 运行所有测试
    pytest testcases/ -v
    
    # 运行指定文件
    pytest testcases/test_login.py -v
    
    # 运行指定类
    pytest testcases/test_login.py::TestLogin -v
    
    # 运行指定用例
    pytest testcases/test_login.py::TestLogin::test_login_success -v
    
    # 按标签运行
    pytest -m smoke -v
    

### 生成报告 
    
    
    # HTML 报告
    pytest testcases/ --html=report.html --self-contained-html
    
    # Allure 报告（更美观）
    pytest testcases/ --alluredir=allure-results
    allure serve allure-results
    

### pytest.ini 配置 
    
    
    [pytest]
    testpaths = testcases
    python_files = test_*.py
    python_classes = Test*
    python_functions = test_*
    markers =
        smoke: 冒烟测试
        regression: 回归测试
    addopts = -v --tb=short
    

* * *

## 七、CI 集成（Jenkins） 

### Jenkinsfile 
    
    
    pipeline {
        agent any
        
        stages {
            stage('安装依赖') {
                steps {
                    sh 'pip install -r requirements.txt'
                }
            }
            
            stage('运行测试') {
                steps {
                    sh 'pytest testcases/ --html=report.html --junitxml=report.xml'
                }
                post {
                    always {
                        junit 'report.xml'
                        publishHTML(target: [
                            allowMissing: true,
                            alwaysLinkToLastBuild: true,
                            keepAll: true,
                            reportDir: '.',
                            reportFiles: 'report.html',
                            reportName: '接口测试报告'
                        ])
                    }
                }
            }
        }
    }
    

* * *

## 八、进阶技巧 

### 接口依赖处理 
    
    
    class TestOrderFlow:
        """订单流程测试：下单 → 支付 → 查询"""
        
        def test_order_flow(self, auth_api):
            # 1. 创建订单
            resp = auth_api.post("/orders", json={"product_id": 1, "quantity": 2})
            order_id = resp.json()["data"]["order_id"]
            
            # 2. 支付订单
            resp = auth_api.post(f"/orders/{order_id}/pay", json={"pay_method": "alipay"})
            assert resp.json()["code"] == 0
            
            # 3. 查询订单状态
            resp = auth_api.get(f"/orders/{order_id}")
            assert resp.json()["data"]["status"] == "paid"
    

### 接口签名处理 
    
    
    import hashlib
    import time
    
    
    def generate_sign(params, secret):
        """生成接口签名"""
        sorted_params = sorted(params.items())
        sign_str = "&".join(f"{k}={v}" for k, v in sorted_params)
        sign_str += f"&secret={secret}"
        return hashlib.md5(sign_str.encode()).hexdigest()
    
    
    # 在 BaseAPI 中使用
    class SignedAPI(BaseAPI):
        def request(self, method, path, **kwargs):
            params = kwargs.get("params", {})
            params["timestamp"] = int(time.time())
            params["sign"] = generate_sign(params, "your_secret")
            kwargs["params"] = params
            return super().request(method, path, **kwargs)
    

* * *

_接口自动化不难，难的是坚持维护。一个好的测试框架，代码结构清晰、断言明确、数据分离。从第一个接口开始，慢慢积累，你会越来越得心应手。_