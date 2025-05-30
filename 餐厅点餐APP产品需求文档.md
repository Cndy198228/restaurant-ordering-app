# 餐厅点餐APP产品需求文档

## 项目概述

### 产品定位
面向西班牙地区中国餐厅的移动点餐管理系统，提供简洁高效的点餐、收银和数据管理功能。

### 目标用户
- 主要用户：中国餐厅服务员、收银员
- 使用场景：餐厅内点餐、结账、订单管理
- 使用环境：西班牙，中文操作界面

## 核心功能需求

### 1. 菜品管理模块
**功能描述：** 餐厅管理员可以添加、编辑、删除菜品信息

**具体需求：**
- 菜品名称录入（支持中文）
- 菜品单价设置（欧元计价）
- 菜品分类管理（如：主食、汤类、饮品等）
- 菜品状态管理（可售/售罄）
- 菜品图片上传（可选）

**界面设计要求：**
- 简洁的列表展示
- 快速添加按钮
- 搜索功能
- 批量操作功能

### 2. 点餐下单模块
**功能描述：** 服务员为顾客点餐并生成订单

**具体需求：**
- 菜品快速选择界面
- 数量调整功能
- 订单实时计算总价
- 备注信息添加
- 桌号标记
- 结账方式选择（现金/刷卡）

**界面设计要求：**
- 大按钮设计，便于快速操作
- 清晰的价格显示
- 订单预览功能
- 一键清空/撤销功能

### 3. 打印管理模块
**功能描述：** 支持多台小票打印机管理和打印

**具体需求：**
- 支持蓝牙和WiFi连接的小票打印机
- 多打印机设备管理
- 打印模板自定义
- 打印队列管理
- 打印失败重试机制

**技术要求：**
- 支持ESC/POS指令集
- 自动设备发现和连接
- 打印状态监控

### 4. 销售数据查询模块
**功能描述：** 提供销售统计和数据分析功能

**具体需求：**
- 日/周/月销售统计
- 热销菜品排行
- 支付方式统计
- 营业额趋势图表
- 数据导出功能（Excel/PDF）

**界面设计要求：**
- 直观的图表展示
- 时间范围筛选
- 数据钻取功能

### 5. 订单管理模块
**功能描述：** 管理历史订单，支持最多500条记录

**具体需求：**
- 订单历史查询
- 订单状态管理（待处理/已完成/已取消）
- 订单详情查看
- 订单修改和退款
- 自动删除最旧记录（超过500条时）

**数据管理策略：**
- 采用FIFO（先进先出）原则
- 自动清理机制
- 重要订单标记保护

## 技术架构需求

### 1. 前端技术栈
- **开发平台：** Android原生应用
- **开发语言：** Kotlin
- **最低支持版本：** Android 7.0 (API Level 24)
- **架构模式：** MVVM + Repository Pattern
- **UI框架：** Material Design 3

### 2. 后端架构
- **云服务提供商：** Amazon Web Services (AWS)
- **计算服务：** AWS Lambda (无服务器架构)
- **API网关：** Amazon API Gateway
- **数据库：** Amazon RDS (MySQL 8.0)
- **文件存储：** Amazon S3
- **身份认证：** AWS Cognito

### 3. 数据库设计

#### 菜品表 (dishes)
```sql
CREATE TABLE dishes (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL COMMENT '菜品名称',
    price DECIMAL(10,2) NOT NULL COMMENT '价格(欧元)',
    category_id INT COMMENT '分类ID',
    description TEXT COMMENT '菜品描述',
    image_url VARCHAR(255) COMMENT '图片URL',
    status ENUM('available', 'unavailable') DEFAULT 'available',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

#### 订单表 (orders)
```sql
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_number VARCHAR(50) UNIQUE NOT NULL COMMENT '订单号',
    table_number VARCHAR(20) COMMENT '桌号',
    total_amount DECIMAL(10,2) NOT NULL COMMENT '总金额',
    payment_method ENUM('cash', 'card') NOT NULL COMMENT '支付方式',
    status ENUM('pending', 'completed', 'cancelled') DEFAULT 'pending',
    notes TEXT COMMENT '备注',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

#### 订单明细表 (order_items)
```sql
CREATE TABLE order_items (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT NOT NULL,
    dish_id INT NOT NULL,
    quantity INT NOT NULL DEFAULT 1,
    unit_price DECIMAL(10,2) NOT NULL,
    subtotal DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (dish_id) REFERENCES dishes(id)
);
```

#### 打印机配置表 (printers)
```sql
CREATE TABLE printers (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL COMMENT '打印机名称',
    type ENUM('bluetooth', 'wifi') NOT NULL COMMENT '连接类型',
    address VARCHAR(100) NOT NULL COMMENT 'MAC地址或IP地址',
    status ENUM('online', 'offline') DEFAULT 'offline',
    is_default BOOLEAN DEFAULT FALSE COMMENT '是否默认打印机',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 4. 数据管理策略

#### 500条订单限制实现
```sql
-- 创建触发器，自动删除最旧的订单
DELIMITER //
CREATE TRIGGER order_limit_trigger 
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    DECLARE order_count INT;
    SELECT COUNT(*) INTO order_count FROM orders;
    
    IF order_count > 500 THEN
        DELETE FROM orders 
        WHERE id IN (
            SELECT id FROM (
                SELECT id FROM orders 
                ORDER BY created_at ASC 
                LIMIT (order_count - 500)
            ) AS old_orders
        );
    END IF;
END//
DELIMITER ;
```

## 非功能性需求

### 1. 性能要求
- **响应时间：** 应用启动时间 < 3秒，页面切换 < 1秒
- **并发处理：** 支持同时处理10个订单
- **数据同步：** 本地与云端数据同步延迟 < 5秒
- **离线支持：** 网络断开时可继续使用核心功能

### 2. 可用性要求
- **系统可用性：** 99.5%以上
- **数据备份：** 每日自动备份
- **故障恢复：** 系统故障后1小时内恢复

### 3. 安全性要求
- **数据加密：** 传输和存储数据均采用加密
- **访问控制：** 基于角色的权限管理
- **审计日志：** 记录所有关键操作

### 4. 兼容性要求
- **设备兼容：** 支持7寸以上Android平板和手机
- **打印机兼容：** 支持主流ESC/POS协议打印机
- **网络环境：** 支持WiFi和4G网络

## UI/UX设计要求

### 1. 设计原则
- **简洁性：** 界面简洁明了，避免复杂操作
- **一致性：** 统一的视觉风格和交互模式
- **效率性：** 常用功能一键直达
- **本地化：** 符合中国用户使用习惯

### 2. 界面布局

#### 主界面设计
```
┌─────────────────────────────────────┐
│  餐厅点餐系统           [设置] [统计] │
├─────────────────────────────────────┤
│                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │  点餐   │  │ 菜品管理 │  │ 订单查询 │ │
│  │  下单   │  │         │  │         │ │
│  └─────────┘  └─────────┘  └─────────┘ │
│                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ 打印设置 │  │ 销售统计 │  │ 系统设置 │ │
│  │         │  │         │  │         │ │
│  └─────────┘  └─────────┘  └─────────┘ │
│                                     │
├─────────────────────────────────────┤
│ 今日营业额: €1,234.56    订单: 45笔  │
└─────────────────────────────────────┘
```

#### 点餐界面设计
```
┌─────────────────────────────────────┐
│ [返回] 点餐下单              桌号: 5  │
├─────────────────────────────────────┤
│ 分类: [全部] [主食] [汤类] [饮品]     │
├─────────────────────────────────────┤
│ ┌─────────────────┐ ┌─────────────────┐ │
│ │ 宫保鸡丁        │ │ 麻婆豆腐        │ │
│ │ €12.50    [+]  │ │ €10.00    [+]  │ │
│ └─────────────────┘ └─────────────────┘ │
│ ┌─────────────────┐ ┌─────────────────┐ │
│ │ 糖醋里脊        │ │ 青椒肉丝        │ │
│ │ €15.00    [+]  │ │ €11.50    [+]  │ │
│ └─────────────────┘ └─────────────────┘ │
├─────────────────────────────────────┤
│ 购物车 (3项)              总计: €38.00 │
│ [查看详情] [清空] [现金结账] [刷卡结账] │
└─────────────────────────────────────┘
```

### 3. 交互设计
- **手势支持：** 滑动、点击、长按
- **反馈机制：** 操作确认、加载状态、错误提示
- **快捷操作：** 常用功能快捷键

## 开发计划

### 第一阶段：需求分析与设计 (2周)
- 详细需求调研
- UI/UX设计
- 技术架构设计
- 数据库设计

### 第二阶段：基础架构搭建 (2周)
- AWS云服务环境搭建
- Android项目初始化
- 基础框架搭建
- 数据库创建和配置

### 第三阶段：核心功能开发 (4周)
- 菜品管理模块
- 点餐下单模块
- 基础打印功能
- 数据同步机制

### 第四阶段：高级功能开发 (2周)
- 多打印机支持
- 销售统计功能
- 订单管理优化
- 性能优化

### 第五阶段：测试与部署 (2周)
- 功能测试
- 性能测试
- 用户验收测试
- 正式发布

## 技术选型建议

### 1. 开发工具
- **IDE：** Android Studio
- **版本控制：** Git + GitHub
- **项目管理：** Jira 或 Trello
- **设计工具：** Figma

### 2. 核心库和框架
```gradle
// Android核心依赖
implementation 'androidx.core:core-ktx:1.12.0'
implementation 'androidx.appcompat:appcompat:1.6.1'
implementation 'com.google.android.material:material:1.11.0'

// 架构组件
implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0'
implementation 'androidx.lifecycle:lifecycle-livedata-ktx:2.7.0'
implementation 'androidx.navigation:navigation-fragment-ktx:2.7.6'

// 数据库
implementation 'androidx.room:room-runtime:2.6.1'
implementation 'androidx.room:room-ktx:2.6.1'
kapt 'androidx.room:room-compiler:2.6.1'

// 网络请求
implementation 'com.squareup.retrofit2:retrofit:2.9.0'
implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
implementation 'com.squareup.okhttp3:logging-interceptor:4.12.0'

// 依赖注入
implementation 'com.google.dagger:hilt-android:2.48'
kapt 'com.google.dagger:hilt-compiler:2.48'

// 图片加载
implementation 'com.github.bumptech.glide:glide:4.16.0'

// 图表库
implementation 'com.github.PhilJay:MPAndroidChart:v3.1.0'

// 蓝牙打印
implementation 'com.github.DantSu:ESCPOS-ThermalPrinter-Android:3.3.0'
```

### 3. AWS服务配置
- **Lambda函数：** Node.js 18.x 运行时
- **RDS配置：** db.t3.micro 实例，20GB存储
- **S3存储：** 标准存储类，生命周期管理
- **API Gateway：** REST API，CORS配置

## 安全考虑

### 1. 数据安全
- **传输加密：** HTTPS/TLS 1.3
- **存储加密：** AES-256加密
- **密钥管理：** AWS KMS

### 2. 访问控制
- **API认证：** JWT Token
- **权限管理：** 基于角色的访问控制
- **会话管理：** 自动过期和刷新

### 3. 数据备份
- **自动备份：** 每日增量备份
- **异地备份：** 多区域数据复制
- **恢复测试：** 定期恢复演练

## 运维和维护

### 1. 监控告警
- **应用性能监控：** AWS CloudWatch
- **错误追踪：** Crashlytics
- **用户行为分析：** Google Analytics

### 2. 日志管理
- **应用日志：** 结构化日志记录
- **系统日志：** CloudWatch Logs
- **审计日志：** 操作记录和追踪

### 3. 版本管理
- **灰度发布：** 分批次用户发布
- **回滚机制：** 快速版本回退
- **热修复：** 紧急bug修复

## 成本估算

### 1. 开发成本
- **人力成本：** €45,000 - €60,000
- **云服务成本：** €200 - €500/月
- **第三方服务：** €100 - €300/月
- **设备测试：** €2,000 - €3,000

### 2. 运营成本
- **服务器费用：** €300 - €800/月
- **维护支持：** €500 - €1,000/月
- **功能迭代：** €2,000 - €5,000/季度

## 风险评估

### 1. 技术风险
- **打印机兼容性：** 中等风险，需要充分测试
- **网络稳定性：** 低风险，有离线模式支持
- **性能瓶颈：** 低风险，云服务可扩展

### 2. 业务风险
- **用户接受度：** 中等风险，需要用户培训
- **竞争压力：** 中等风险，需要差异化功能
- **法规合规：** 低风险，遵循GDPR要求

### 3. 项目风险
- **进度延期：** 中等风险，预留缓冲时间
- **成本超支：** 低风险，分阶段控制
- **团队变动：** 中等风险，知识文档化

## 成功标准

### 1. 功能完整性
- 所有核心功能正常运行
- 用户验收测试通过
- 性能指标达到要求

### 2. 用户满意度
- 用户培训完成率 > 90%
- 用户满意度评分 > 4.0/5.0
- 日活跃用户 > 80%

### 3. 技术指标
- 系统可用性 > 99.5%
- 平均响应时间 < 2秒
- 崩溃率 < 0.1%

---

**文档版本：** v1.0  
**创建日期：** 2024年  
**最后更新：** 2024年  
**文档状态：** 草案