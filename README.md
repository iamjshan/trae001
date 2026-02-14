# 标准物质管理助手

一个基于 HTML/CSS/JavaScript + Supabase 的标准物质库存管理 Web 应用。

## 功能特性

- 📦 **库存管理** - 标准物质入库、出库、库存查询
- 📷 **AI 智能识别** - 拍照/上传图片，AI 自动提取标签信息
- 📄 **PDF 支持** - 支持上传和预览 PDF 文档
- 📊 **数据统计** - 库存统计、临期预警、状态分布图表
- 🔍 **扫码功能** - 二维码扫描快速出库
- 📱 **响应式设计** - 支持手机、平板、电脑访问
- 💾 **离线支持** - IndexedDB 本地缓存，快速加载
- 📤 **数据导入导出** - Excel 格式导入导出

## 技术栈

- **前端**: HTML5 + Tailwind CSS + Font Awesome
- **后端**: Supabase (PostgreSQL + Auth + Storage)
- **AI 识别**: 豆包大模型 (doubao-seed-1-6-vision)
- **图表**: Chart.js
- **二维码**: qrcode.js + jsQR
- **PDF**: PDF.js
- **Excel**: SheetJS

## 快速开始

### 1. 配置 Supabase

#### 1.1 创建项目

1. 访问 [Supabase](https://supabase.com) 创建新项目
2. 记录项目 URL 和 API Key

#### 1.2 配置 API Key

在 `index.html` 中修改以下配置：

```javascript
// ==================== Supabase 配置 ====================
const SUPABASE_URL = 'https://your-project.supabase.co'
const SUPABASE_ANON_KEY = 'sb_publishable_7D5tnogliJUvQ_Pp2ASMwQ__RmVrE-T'
```

#### 1.3 执行 SQL 初始化

在 Supabase SQL Editor 中执行以下代码：

```sql
-- ============================================
-- 标准物质管理系统 - 数据库初始化脚本
-- ============================================

-- 1. 创建库存表
CREATE TABLE IF NOT EXISTS stock_items (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    name TEXT NOT NULL,
    code TEXT,
    batch_number TEXT,
    manufacturer TEXT,
    concentration TEXT,
    unit TEXT,
    uncertainty TEXT,
    storage_condition TEXT,
    dilution_condition TEXT,
    expiry_date DATE,
    quantity INTEGER DEFAULT 1,
    unique_id TEXT UNIQUE,
    status TEXT DEFAULT 'normal',
    images JSONB DEFAULT '[]',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 2. 创建操作记录表
CREATE TABLE IF NOT EXISTS operation_records (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    type TEXT NOT NULL, -- 'in', 'out', 'warning'
    material_name TEXT NOT NULL,
    material_id UUID REFERENCES stock_items(id),
    quantity INTEGER DEFAULT 1,
    operator TEXT,
    operator_name TEXT,
    notes TEXT,
    images JSONB DEFAULT '[]',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 3. 创建用户资料表
CREATE TABLE IF NOT EXISTS profiles (
    id UUID REFERENCES auth.users(id) PRIMARY KEY,
    name TEXT,
    role TEXT DEFAULT 'user', -- 'admin', 'user'
    phone TEXT,
    department TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 4. 创建更新时间触发器
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- 库存表更新触发器
DROP TRIGGER IF EXISTS update_stock_items_updated_at ON stock_items;
CREATE TRIGGER update_stock_items_updated_at
    BEFORE UPDATE ON stock_items
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- 用户资料表更新触发器
DROP TRIGGER IF EXISTS update_profiles_updated_at ON profiles;
CREATE TRIGGER update_profiles_updated_at
    BEFORE UPDATE ON profiles
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- 5. 设置 RLS (行级安全) 策略

-- 库存表策略
ALTER TABLE stock_items ENABLE ROW LEVEL SECURITY;

DROP POLICY IF EXISTS "Allow all operations" ON stock_items;
CREATE POLICY "Allow all operations" ON stock_items
    FOR ALL
    USING (true)
    WITH CHECK (true);

-- 操作记录表策略
ALTER TABLE operation_records ENABLE ROW LEVEL SECURITY;

DROP POLICY IF EXISTS "Allow all operations" ON operation_records;
CREATE POLICY "Allow all operations" ON operation_records
    FOR ALL
    USING (true)
    WITH CHECK (true);

-- 用户资料表策略
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

DROP POLICY IF EXISTS "Users can view all profiles" ON profiles;
CREATE POLICY "Users can view all profiles" ON profiles
    FOR SELECT
    USING (true);

DROP POLICY IF EXISTS "Users can update own profile" ON profiles;
CREATE POLICY "Users can update own profile" ON profiles
    FOR UPDATE
    USING (auth.uid() = id);

DROP POLICY IF EXISTS "Users can insert own profile" ON profiles;
CREATE POLICY "Users can insert own profile" ON profiles
    FOR INSERT
    WITH CHECK (auth.uid() = id);

-- 6. 创建新用户自动创建资料触发器
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO public.profiles (id, name, role)
    VALUES (
        NEW.id,
        COALESCE(NEW.raw_user_meta_data->>'name', NEW.email),
        COALESCE(NEW.raw_user_meta_data->>'role', 'user')
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- 触发器
DROP TRIGGER IF EXISTS on_auth_user_created ON auth.users;
CREATE TRIGGER on_auth_user_created
    AFTER INSERT ON auth.users
    FOR EACH ROW
    EXECUTE FUNCTION public.handle_new_user();

-- ============================================
-- 初始化完成
-- ============================================
```

### 2. 配置 AI 识别（可选）

如需使用 AI 智能识别功能，在 `index.html` 中配置豆包 API：

```javascript
// ==================== 豆包 AI 配置 ====================
const DOUBAO_CONFIG = {
    MODEL: 'doubao-seed-1-6-vision-250815',
    ENDPOINT: 'https://ark.cn-beijing.volces.com/api/v3/chat/completions',
    API_KEY: 'your-doubao-api-key'  // 替换为你的 API Key
}
```

获取 API Key：
1. 访问 [火山方舟](https://www.volcengine.com/product/ark)
2. 创建应用并获取 API Key

### 3. 部署使用

#### 方式一：直接打开
双击 `index.html` 文件在浏览器中打开（部分功能受限）

#### 方式二：本地服务器
```bash
# 使用 Python 启动本地服务器
python -m http.server 8000

# 或使用 Node.js
npx serve .
```

然后访问 `http://localhost:8000`

#### 方式三：部署到静态托管
将文件上传到：
- Vercel
- Netlify
- GitHub Pages
- 或任何静态文件托管服务

## 使用说明

### 首次使用

1. 打开应用，进入登录页面
2. 点击"注册账号"创建管理员账号
3. 登录后进入系统

### 入库操作

1. 点击首页"入库"按钮
2. 填写标准物质信息，或点击"AI识别"拍照自动提取
3. 上传相关图片/PDF（可选）
4. 点击保存

### 出库操作

1. 进入"库存管理"页面
2. 找到要出库的物品，点击"出库"按钮
3. 填写出库信息，上传图片（可选）
4. 点击确认出库

### 扫码出库

1. 点击首页"扫码"按钮
2. 扫描标准物质标签上的二维码
3. 确认信息后点击出库

## 数据备份

### 导出数据
进入"系统设置"页面，点击"导出数据"按钮，可将所有数据导出为 Excel 文件。

### 导入数据
进入"系统设置"页面，点击"导入数据"按钮，选择之前导出的 Excel 文件。

## 项目结构

```
.
├── index.html          # 主应用文件（包含所有 HTML/CSS/JS）
├── README.md           # 项目说明文档
└── supabase_schema.sql # 数据库初始化脚本
```

## 浏览器兼容性

- Chrome 80+
- Firefox 75+
- Safari 13+
- Edge 80+

## 注意事项

1. **浏览器存储**: 应用使用 IndexedDB 缓存数据，请勿在隐私模式下使用
2. **图片存储**: 图片以 Base64 格式存储，建议单张图片不超过 5MB
3. **网络要求**: 首次加载需要网络连接，后续可使用本地缓存
4. **Supabase 限制**: 免费版有数据库大小和请求次数限制

## 常见问题

### Q: 登录后页面空白？
A: 检查 Supabase 配置是否正确，以及 SQL 脚本是否已执行。

### Q: AI 识别失败？
A: 检查豆包 API Key 是否配置正确，以及网络连接是否正常。

### Q: 图片上传失败？
A: 检查图片大小是否超过限制，或尝试压缩图片后重新上传。

### Q: 数据加载很慢？
A: 首次加载需要从云端同步数据，后续会从本地缓存加载。如持续很慢，请检查网络连接。

## 技术细节

### 数据同步策略

1. **本地优先**: 优先从 IndexedDB 加载数据，页面立即显示
2. **后台同步**: 页面显示后，后台自动同步云端最新数据
3. **增量更新**: 只同步变化的数据，减少网络请求

### 离线支持

- 使用 IndexedDB 存储数据（容量约 50MB+）
- 支持完全离线模式（需在设置中开启）
- 离线模式下数据保存在本地，联网后自动同步

## 更新日志

### v1.0.0
- ✨ 初始版本发布
- 📦 库存管理功能
- 📷 AI 智能识别
- 📄 PDF 支持
- 📊 数据统计图表
- 🔍 二维码扫描

## 许可证

MIT License

## 联系方式

如有问题或建议，欢迎反馈。

---

**注意**: 请妥善保管您的 Supabase API Key，不要将其提交到公共代码仓库。
