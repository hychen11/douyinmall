# DataBase Design

### Mysql

`/etc/mysql/my.cnf`

```bash
[mysqld]
# 字符集设置
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci

# 连接数设置,允许的最大并发连接数 为 200
max_connections=200

# 缓存设置
innodb_buffer_pool_size=1G

# 日志设置
slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=2  
```

`utf8mb4` 是完整的 UTF-8 

`utf8mb4_unicode_ci`排序规则，支持中文排序

`slow_query_log` 启用慢查询日志，用于记录执行时间过长的 SQL 语句，帮助数据库优化

` long_query_time=2` **执行时间超过 2 秒的 SQL** 将被记录

### Design Principle

1. 避免使用外键约束
   - 提高性能
   - 简化维护
   - 便于分库分表
2. 必要的字段设计
   - 主键使用`BIGINT`类型
   - 包含创建和更新时间
   - 使用软删除
   - 状态字段的规范设计
3. 索引设计
   - 经常查询的字段建立索引
   - 避免过度索引
   - 考虑组合索引的顺序

### Create DataBase &Table

`mysql -u root -p`

```sql
-- 创建数据库
CREATE DATABASE IF NOT EXISTS douyin_mall DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;

-- 选择数据库
USE douyin_mall;
```

这里password用**bcrypt 哈希算法** 进行了加密

#### **1. 用户管理**

```sql
-- Users table
CREATE TABLE users (
   id BIGINT PRIMARY KEY AUTO_INCREMENT,
   username VARCHAR(50) NOT NULL UNIQUE,
   password VARCHAR(255) NOT NULL,
   email VARCHAR(100) NOT NULL UNIQUE,
   phone VARCHAR(20),
   avatar_url VARCHAR(255),
   role ENUM('user', 'admin') DEFAULT 'user',
   status TINYINT DEFAULT 1 COMMENT '1: active, 0: inactive, -1: deleted',
   created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
   updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
   deleted_at TIMESTAMP NULL,
   INDEX idx_email (email),
   INDEX idx_phone (phone),
   INDEX idx_status_deleted (status, deleted_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;


-- User addresses table
CREATE TABLE user_addresses (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    recipient_name VARCHAR(50) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    province VARCHAR(50) NOT NULL,
    city VARCHAR(50) NOT NULL,
    district VARCHAR(50) NOT NULL,
    detailed_address VARCHAR(200) NOT NULL,
    is_default BOOLEAN DEFAULT FALSE,
    status TINYINT DEFAULT 1 COMMENT '1: active, 0: inactive',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL,
    INDEX idx_user_id (user_id),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### **2. 商品管理**

```sql
-- Product categories table
CREATE TABLE categories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    parent_id BIGINT,
    level TINYINT NOT NULL,
    sort_order INT DEFAULT 0,
    status TINYINT DEFAULT 1 COMMENT '1: active, 0: inactive',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL,
    INDEX idx_parent_id (parent_id),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Products table
CREATE TABLE products (
      id BIGINT PRIMARY KEY AUTO_INCREMENT,
      category_id BIGINT NOT NULL,
      name VARCHAR(100) NOT NULL,
      description TEXT,
      price DECIMAL(10,2) NOT NULL,
      original_price DECIMAL(10,2),
      stock INT NOT NULL DEFAULT 0,
      images JSON,
      sales_count INT DEFAULT 0,
      status TINYINT DEFAULT 1 COMMENT '1: on sale, 0: off sale, -1: deleted',
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
      deleted_at TIMESTAMP NULL,
      INDEX idx_category_id (category_id),
      INDEX idx_status (status),
      INDEX idx_sales (sales_count),
      INDEX idx_updated_at (updated_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### **3. 订单管理**

```sql
-- Orders table
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_no VARCHAR(50) NOT NULL UNIQUE,
    user_id BIGINT NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    actual_amount DECIMAL(10,2) NOT NULL,
    address_snapshot JSON NOT NULL COMMENT 'Snapshot of address at order time',
    status TINYINT NOT NULL DEFAULT 0 COMMENT '0: pending payment, 1: paid, 2: shipped, 3: delivered, 4: completed, -1: cancelled',
    payment_type TINYINT COMMENT '1: alipay, 2: wechat, 3: credit card',
    payment_time TIMESTAMP NULL,
    shipping_time TIMESTAMP NULL,
    delivery_time TIMESTAMP NULL,
    completion_time TIMESTAMP NULL,
    cancel_time TIMESTAMP NULL,
    cancel_reason VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id),
    INDEX idx_order_no (order_no),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Order items table
CREATE TABLE order_items (
     id BIGINT PRIMARY KEY AUTO_INCREMENT,
     order_id BIGINT NOT NULL,
     product_id BIGINT NOT NULL,
     product_snapshot JSON NOT NULL COMMENT 'Snapshot of product at order time',
     quantity INT NOT NULL,
     price DECIMAL(10,2) NOT NULL,
     created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
     INDEX idx_order_id (order_id),
     INDEX idx_product_id (product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### **4. 购物车管理**

```sql
-- Shopping cart table
CREATE TABLE shopping_cart_items (
     id BIGINT PRIMARY KEY AUTO_INCREMENT,
     user_id BIGINT NOT NULL,
     product_id BIGINT NOT NULL,
     quantity INT NOT NULL DEFAULT 1,
     selected BOOLEAN DEFAULT TRUE,
     status TINYINT DEFAULT 1 COMMENT '1: valid, 0: invalid',
     created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
     updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
     UNIQUE KEY uk_user_product (user_id, product_id, status),
     INDEX idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### **5. 支付管理**

```sql
-- Product reviews table
CREATE TABLE product_reviews (
     id BIGINT PRIMARY KEY AUTO_INCREMENT,
     user_id BIGINT NOT NULL,
     product_id BIGINT NOT NULL,
     order_id BIGINT NOT NULL,
     rating TINYINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
     content TEXT,
     images JSON,
     status TINYINT DEFAULT 1 COMMENT '1: visible, 0: hidden',
     created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
     updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
     deleted_at TIMESTAMP NULL,
     INDEX idx_product_id (product_id),
     INDEX idx_user_id (user_id),
     INDEX idx_order_id (order_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Payment records table
CREATE TABLE payment_records (
     id BIGINT PRIMARY KEY AUTO_INCREMENT,
     order_id BIGINT NOT NULL,
     payment_no VARCHAR(100) NOT NULL UNIQUE,
     transaction_id VARCHAR(100),
     amount DECIMAL(10,2) NOT NULL,
     payment_type TINYINT NOT NULL COMMENT '1: alipay, 2: wechat, 3: credit card',
     status TINYINT NOT NULL DEFAULT 0 COMMENT '0: pending, 1: success, 2: failed, 3: refunded',
     callback_time TIMESTAMP NULL COMMENT 'Payment callback time',
     callback_data JSON COMMENT 'Complete callback data',
     created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
     updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
     INDEX idx_order_id (order_id),
     INDEX idx_payment_no (payment_no),
     INDEX idx_transaction_id (transaction_id),
     INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Insert Data

#### 1 插入用户管理数据

```sql
INSERT INTO users (username, password, email, phone, avatar_url, role, status) VALUES
    ('zhangsan', '$2a$10$1qAz2wSx3eDc4rFv5tGb5t', 'zhangsan@example.com', '13800138001', 'https://example.com/avatars/1.jpg', 'user', 1),
    ('lisi', '$2a$10$2qAz2wSx3eDc4rFv5tGb5u', 'lisi@example.com', '13800138002', 'https://example.com/avatars/2.jpg', 'user', 1),
    ('wangwu', '$2a$10$3qAz2wSx3eDc4rFv5tGb5v', 'wangwu@example.com', '13800138003', 'https://example.com/avatars/3.jpg', 'user', 1),
    ('admin', '$2a$10$4qAz2wSx3eDc4rFv5tGb5w', 'admin@example.com', '13800138004', 'https://example.com/avatars/4.jpg', 'admin', 1),
    ('zhouwen', '$2a$10$5qAz2wSx3eDc4rFv5tGb5x', 'zhouwen@example.com', '13800138005', 'https://example.com/avatars/5.jpg', 'user', 1),
    ('sunqi', '$2a$10$6qAz2wSx3eDc4rFv5tGb5y', 'sunqi@example.com', '13800138006', 'https://example.com/avatars/6.jpg', 'user', 1),
    ('liuba', '$2a$10$7qAz2wSx3eDc4rFv5tGb5z', 'liuba@example.com', '13800138007', 'https://example.com/avatars/7.jpg', 'user', 1),
    ('huling', '$2a$10$8qAz2wSx3eDc4rFv5tGb5a', 'huling@example.com', '13800138008', 'https://example.com/avatars/8.jpg', 'user', 0);
    
INSERT INTO user_addresses (user_id, recipient_name, phone, province, city, district, detailed_address, is_default, status) VALUES
    (1, '张三', '13800138001', '北京市', '北京市', '朝阳区', '建国路88号SOHO大厦A座1001室', TRUE, 1),
    (1, '张三', '13800138001', '北京市', '北京市', '海淀区', '中关村东路66号', FALSE, 1),
    (2, '李四', '13800138002', '上海市', '上海市', '浦东新区', '世纪大道200号', TRUE, 1),
    (3, '王五', '13800138003', '广东省', '广州市', '天河区', '珠江新城88号', TRUE, 1),
    (4, 'Admin', '13800138004', '江苏省', '南京市', '秦淮区', '夫子庙路100号', TRUE, 1),
    (5, '周文', '13800138005', '四川省', '成都市', '武侯区', '高新区天府大道1000号', TRUE, 1),
    (6, '孙七', '13800138006', '浙江省', '杭州市', '西湖区', '西湖风景区湖滨路9号', TRUE, 1),
    (7, '刘八', '13800138007', '福建省', '厦门市', '思明区', '鼓浪屿龙头路33号', TRUE, 1),
    (8, '胡玲', '13800138008', '湖南省', '长沙市', '岳麓区', '岳麓山路77号', FALSE, 0);

```

#### 2 插入商品管理

```sql
INSERT INTO categories (name, parent_id, level, sort_order, status) VALUES
    ('电子产品', NULL, 1, 1, 1),
    ('手机', 1, 2, 1, 1),
    ('笔记本电脑', 1, 2, 2, 1),
    ('家用电器', NULL, 1, 2, 1),
    ('电视', 4, 2, 1, 1),
    ('洗衣机', 4, 2, 2, 1),
    ('服装', NULL, 1, 3, 1),
    ('男装', 7, 2, 1, 1),
    ('女装', 7, 2, 2, 1);

INSERT INTO products (category_id, name, description, price, original_price, stock, images, sales_count, status) VALUES
    (2, 'iPhone 14 Pro', '最新苹果旗舰手机，搭载A16芯片，超强影像能力', 8999.00, 9999.00, 50, 
     '["https://example.com/images/iphone14pro-1.jpg", "https://example.com/images/iphone14pro-2.jpg"]', 1000, 1),
     
    (2, 'Samsung Galaxy S23 Ultra', '三星顶级旗舰，2亿像素，S Pen', 7999.00, 8999.00, 30, 
     '["https://example.com/images/s23ultra-1.jpg", "https://example.com/images/s23ultra-2.jpg"]', 800, 1),
     
    (3, 'MacBook Pro 16', '苹果高性能笔记本，M2 Max芯片', 18999.00, 19999.00, 20, 
     '["https://example.com/images/macbook16-1.jpg", "https://example.com/images/macbook16-2.jpg"]', 300, 1),
     
    (3, 'Dell XPS 15', '戴尔超薄高端笔记本，OLED屏幕', 12999.00, 13999.00, 15, 
     '["https://example.com/images/dellxps15-1.jpg", "https://example.com/images/dellxps15-2.jpg"]', 200, 1),
     
    (5, 'Sony 65寸 OLED 4K电视', '索尼旗舰OLED电视，超高清画质', 14999.00, 15999.00, 10, 
     '["https://example.com/images/sony-tv-1.jpg", "https://example.com/images/sony-tv-2.jpg"]', 500, 1),
     
    (6, '海尔滚筒洗衣机', '智能变频，节能省水', 2999.00, 3299.00, 25, 
     '["https://example.com/images/haier-washer-1.jpg"]', 1200, 1),
     
    (8, '男士商务西装', '高端西服，适合商务场合', 1299.00, 1499.00, 100, 
     '["https://example.com/images/mens-suit-1.jpg"]', 400, 1),
     
    (9, '女士时尚连衣裙', '高端时尚，优雅大方', 799.00, 999.00, 150, 
     '["https://example.com/images/womens-dress-1.jpg"]', 600, 1);

```

#### 3 插入订单和购物车数据

```sql
-- 插入购物车项目
INSERT INTO shopping_cart_items (user_id, product_id, quantity, selected, status) VALUES
                                                                                      (1, 1, 1, true, 1),
                                                                                      (1, 3, 1, true, 1),
                                                                                      (2, 2, 2, true, 1),
                                                                                      (3, 4, 1, false, 1);

-- 插入订单（包含不同状态的订单）
INSERT INTO orders (order_no, user_id, total_amount, actual_amount, address_snapshot, status, payment_type, payment_time, shipping_time, delivery_time) VALUES
 ('202501180001', 1, 8999.00, 8999.00, '{"recipient_name":"张三","phone":"13800138001","address":"广东省深圳市南山区科技园南区T3栋801"}', 4, 1, '2025-01-18 10:00:00', '2025-01-18 14:00:00', '2025-01-19 10:00:00'),
 ('202501180002', 1, 14999.00, 14999.00, '{"recipient_name":"张三","phone":"13800138001","address":"广东省深圳市南山区科技园南区T3栋801"}', 2, 2, '2025-01-18 11:00:00', '2025-01-18 15:00:00', NULL),                                                               ('202501180003', 2, 9998.00, 9998.00, '{"recipient_name":"李四","phone":"13800138002","address":"广东省广州市天河区天河路222号"}', 1, 1, '2025-01-18 12:00:00', NULL, NULL),                                                                                           ('202501180004', 3, 4699.00, 4699.00, '{"recipient_name":"王五","phone":"13800138003","address":"北京市朝阳区朝阳门外大街19号"}', 0, NULL, NULL, NULL, NULL);

-- 插入订单项
INSERT INTO order_items (order_id, product_id, product_snapshot, quantity, price) VALUES
(1, 1, '{"name":"iPhone 15 Pro 256GB 暗夜紫","price":8999.00}', 1, 8999.00),
(2, 3, '{"name":"MacBook Pro 14寸 M3芯片","price":14999.00}', 1, 14999.00),
(3, 2, '{"name":"小米14 Pro 512GB 钛金黑","price":4999.00}', 2, 4999.00),
(4, 4, '{"name":"iPad Air 5 256GB WIFI版","price":4699.00}', 1, 4699.00);
```

#### 4 支付管理

```sql
-- 插入商品评价
INSERT INTO product_reviews (user_id, product_id, order_id, rating, content, images, status) VALUES
(1, 1, 1, 5, '非常好用的手机，外观设计非常惊艳，性能也很强劲！', '["review1.jpg", "review2.jpg"]', 1),
(1, 3, 2, 4, 'Mac的系统非常流畅，就是价格稍贵', '["review3.jpg"]', 1),
(2, 2, 3, 5, '国产手机性价比之王，拍照效果很赞', '["review4.jpg", "review5.jpg"]', 1);

-- 插入支付记录
INSERT INTO payment_records (order_id, payment_no, transaction_id, amount, payment_type, status, callback_time, callback_data) VALUES
(1, 'PAY202501180001', 'ALIPAY123456', 8999.00, 1, 1, '2025-01-18 10:00:00', '{"trade_no":"ALIPAY123456","buyer_id":"2088123456"}'),
(2, 'PAY202501180002', 'WXPAY123456', 14999.00, 2, 1, '2025-01-18 11:00:00', '{"transaction_id":"WXPAY123456","openid":"wx123456"}'),
(3, 'PAY202501180003', 'ALIPAY123457', 9998.00, 1, 1, '2025-01-18 12:00:00', '{"trade_no":"ALIPAY123457","buyer_id":"2088123457"}');
```

### [常见问题与解决方案](https://github.com/ChanMeng666/douyin-mall-go-template/blob/main/docs/database/%E4%BB%8E%E9%9B%B6%E5%BC%80%E5%A7%8B%E5%88%9B%E5%BB%BA%E7%94%B5%E5%95%86%E7%B3%BB%E7%BB%9F%E6%95%B0%E6%8D%AE%E5%BA%93.md?plain=1)

#### 1. 字符集问题

**问题表现**：中文显示为乱码

**解决方案**：
1. 检查数据库字符集设置：
```sql
SHOW VARIABLES LIKE 'character_set%';
```

2. 修改配置文件my.cnf或my.ini：
```ini
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci

[mysql]
default-character-set=utf8mb4

[client]
default-character-set=utf8mb4
```

3. 修改已有数据库的字符集：
```sql
ALTER DATABASE douyin_mall_go_template CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

#### 2. 权限问题

**问题表现**：Access denied for user 'xxx'@'localhost'

**解决方案**：
1. 检查用户权限：
```sql
SHOW GRANTS FOR 'username'@'localhost';
```

2. 授予必要权限：
```sql
GRANT ALL PRIVILEGES ON douyin_mall_go_template.* TO 'username'@'localhost';
FLUSH PRIVILEGES;
```

#### 3. 连接数限制

**问题表现**：Too many connections

**解决方案**：
1. 查看当前连接数限制：
```sql
SHOW VARIABLES LIKE 'max_connections';
```

2. 修改配置文件增加连接数：
```ini
[mysqld]
max_connections=200
```

#### 4. 性能问题

**问题表现**：查询速度慢

**解决方案**：
1. 检查慢查询日志：
```sql
SHOW VARIABLES LIKE 'slow_query%';
```

2. 优化索引：
```sql
-- 添加常用查询条件的索引
ALTER TABLE products ADD INDEX idx_name_status (name, status);
```

3. 使用EXPLAIN分析查询：
```sql
EXPLAIN SELECT * FROM products WHERE status = 1;
```

#### 5. 数据导入导出问题

**问题表现**：导入大量数据时失败

**解决方案**：
1. 修改配置增加超时时间：
```sql
SET GLOBAL max_allowed_packet=1073741824;
```

2. 使用批量导入工具：
```bash
mysqlimport -u root -p --local douyin_mall_go_template data.sql
```

3. 分批导入数据：
```sql
-- 每次导入一部分数据
INSERT INTO products 
SELECT * FROM temp_products 
LIMIT 1000;
```

#### 6. JSON数据处理

**问题表现**：JSON数据操作错误

**解决方案**：
1. 检查JSON格式：
```sql
SELECT JSON_VALID(column_name) FROM table_name;
```

2. 正确的JSON操作：
```sql
-- 提取JSON字段
SELECT JSON_EXTRACT(address_snapshot, '$.recipient_name') FROM orders;

-- 更新JSON字段
UPDATE products 
SET images = JSON_ARRAY_APPEND(images, '$', 'new_image.jpg');
```

#### 7. 备份和恢复

**定期备份建议**：

1. 创建完整备份：
```bash
mysqldump -u root -p douyin_mall_go_template > backup.sql
```

2. 只备份结构：
```bash
mysqldump -u root -p --no-data douyin_mall_go_template > structure.sql
```

3. 恢复数据：
```bash
mysql -u root -p douyin_mall_go_template < backup.sql
```