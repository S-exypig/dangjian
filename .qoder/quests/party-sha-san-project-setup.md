# 党建沙盘Web系统设计文档

## 1. 项目概述

### 1.1 项目简介
党建沙盘是一个任务式、关卡式的线上学习系统，通过9个递进式关卡帮助用户深入学习党建品牌创建知识。用户需要完成当前关卡的所有任务才能解锁下一关卡，最终系统将根据用户填写的内容生成个人学习报告。

### 1.2 核心功能
- 用户注册登录系统
- 9关卡递进式学习
- 多类型任务完成（文本填写、图片上传）
- 学习进度跟踪
- 个人学习报告生成
- 团体组织验证（后期扩展）

### 1.3 技术选型对比

#### 方案A：Python FastAPI（推荐新手）
- **前端**：Vue 3 + Vite + TypeScript + Element Plus
- **后端**：Python FastAPI
- **数据库**：PostgreSQL + Redis
- **部署**：Docker + Nginx

#### 方案B：Go Gin（推荐有Go基础）
- **前端**：Vue 3 + Vite + TypeScript + Element Plus
- **后端**：Go + Gin + GORM
- **数据库**：PostgreSQL + Redis
- **部署**：Docker + Nginx

## 2. 技术架构

### 2.1 整体架构图

```mermaid
graph TB
    A[用户浏览器] --> B[Nginx反向代理]
    B --> C[Vue3前端应用]
    B --> D[FastAPI后端服务]
    
    D --> E[PostgreSQL主数据库]
    D --> F[Redis缓存/会话]
    D --> G[文件存储系统]
    
    subgraph "前端层"
        C --> C1[用户认证模块]
        C --> C2[关卡管理模块]
        C --> C3[任务执行模块]
        C --> C4[进度跟踪模块]
        C --> C5[报告生成模块]
    end
    
    subgraph "后端层"
        D --> D1[用户服务]
        D --> D2[关卡服务]
        D --> D3[任务服务]
        D --> D4[文件服务]
        D --> D5[报告服务]
    end
```

### 2.2 前后端分离架构优势
- **独立开发**：前后端可以并行开发，提高效率
- **技术栈灵活**：前后端可以选择最适合的技术
- **易于扩展**：后期可以轻松添加移动端或小程序
- **部署灵活**：可以独立部署和扩展

## 3. 前端架构设计

### 3.1 技术栈详解

#### 3.1.1 Vue 3 + Composition API
```javascript
// 推荐使用Composition API，代码更清晰
<script setup lang="ts">
import { ref, reactive, computed } from 'vue'

// 响应式数据
const userInfo = reactive({
  username: '',
  currentLevel: 1,
  completedTasks: []
})

// 计算属性
const progressPercentage = computed(() => {
  return (userInfo.completedTasks.length / totalTasks.value) * 100
})
</script>
```

#### 3.1.2 UI框架选择 - Element Plus
选择理由：
- **官方维护**：Vue 3生态系统的官方UI库
- **组件丰富**：提供表单、上传、进度条等必需组件
- **主题定制**：支持红色、黄色主题定制
- **文档完善**：适合初学者学习

#### 3.1.3 状态管理 - Pinia
```typescript
// stores/user.ts
import { defineStore } from 'pinia'

export const useUserStore = defineStore('user', {
  state: () => ({
    userInfo: null,
    currentLevel: 1,
    unlockedLevels: [1],
    taskProgress: {}
  }),
  
  actions: {
    async login(credentials) {
      // 登录逻辑
    },
    
    unlockNextLevel() {
      this.unlockedLevels.push(this.currentLevel + 1)
    }
  }
})
```

### 3.2 组件架构

#### 3.2.1 组件层次结构
```
App.vue
├── Layout/
│   ├── Header.vue (导航栏)
│   ├── Sidebar.vue (关卡列表)
│   └── Footer.vue
├── Views/
│   ├── Login.vue (登录页)
│   ├── Register.vue (注册页)
│   ├── Dashboard.vue (总览页)
│   ├── LevelDetail.vue (关卡详情)
│   └── Report.vue (报告页)
└── Components/
    ├── TaskCard.vue (任务卡片)
    ├── ProgressBar.vue (进度条)
    ├── FileUpload.vue (文件上传)
    └── TextEditor.vue (文本编辑器)
```

#### 3.2.2 关键组件设计

**TaskCard组件**
```vue
<template>
  <el-card class="task-card" :class="{ completed: task.completed }">
    <template #header>
      <div class="task-header">
        <h3>{{ task.title }}</h3>
        <el-tag v-if="task.completed" type="success">已完成</el-tag>
      </div>
    </template>
    
    <div class="task-content">
      <p>{{ task.description }}</p>
      
      <!-- 文本任务 -->
      <el-input 
        v-if="task.type === 'text'"
        v-model="task.content"
        type="textarea"
        :rows="4"
        placeholder="请输入您的答案"
        @blur="saveTask"
      />
      
      <!-- 图片上传任务 -->
      <FileUpload 
        v-if="task.type === 'image'"
        v-model="task.files"
        @upload-success="saveTask"
      />
    </div>
  </el-card>
</template>
```

### 3.3 路由设计

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  {
    path: '/login',
    name: 'Login',
    component: () => import('@/views/Login.vue')
  },
  {
    path: '/',
    name: 'Dashboard',
    component: () => import('@/views/Dashboard.vue'),
    meta: { requiresAuth: true }
  },
  {
    path: '/level/:id',
    name: 'LevelDetail',
    component: () => import('@/views/LevelDetail.vue'),
    meta: { requiresAuth: true }
  },
  {
    path: '/report',
    name: 'Report',
    component: () => import('@/views/Report.vue'),
    meta: { requiresAuth: true }
  }
]

const router = createRouter({
  history: createWebHistory(),
  routes
})

// 路由守卫
router.beforeEach((to, from, next) => {
  const isAuthenticated = localStorage.getItem('token')
  
  if (to.meta.requiresAuth && !isAuthenticated) {
    next('/login')
  } else {
    next()
  }
})
```

### 3.4 主题定制

#### 3.4.1 Element Plus主题配置
```scss
// styles/theme.scss
:root {
  // 主色调 - 党建红
  --el-color-primary: #d32f2f;
  --el-color-primary-light-3: #e57373;
  --el-color-primary-light-5: #ef5350;
  --el-color-primary-light-7: #ffcdd2;
  --el-color-primary-light-8: #ffebee;
  --el-color-primary-light-9: #fce4ec;
  
  // 辅助色 - 金黄色
  --el-color-warning: #ffc107;
  --el-color-warning-light-3: #ffeb3b;
  
  // 背景渐变
  --background-gradient: linear-gradient(135deg, #d32f2f 0%, #ffc107 100%);
}

// 自定义样式
.main-background {
  background: var(--background-gradient);
  min-height: 100vh;
}

.level-card {
  border-left: 4px solid var(--el-color-primary);
  box-shadow: 0 4px 12px rgba(211, 47, 47, 0.15);
}
```

## 4. 后端架构设计

### 4.1 后端技术方案对比

#### 4.1.1 Python FastAPI 方案

**优势：**
- **学习门槛低**：Python语法简单，适合初学者
- **开发速度快**：语法简洁，快速原型开发
- **自动文档**：自动生成OpenAPI/Swagger文档
- **类型安全**：Pydantic提供数据验证和序列化
- **生态丰富**：机器学习、数据分析库丰富
- **异步支持**：原生支持异步编程

**劣势：**
- **运行性能**：相比Go略慢（但对本项目足够）
- **部署复杂度**：需要Python环境和依赖管理
- **内存占用**：相比Go较高

#### 4.1.2 Go Gin 方案

**优势：**
- **极高性能**：编译型语言，运行速度快
- **并发处理**：goroutine轻量级并发，适合高并发
- **部署简单**：编译成单一可执行文件
- **内存效率**：内存占用低，垃圾回收效率高
- **类型安全**：静态类型系统，编译时发现错误
- **跨平台**：一次编译，多平台运行

**劣势：**
- **学习曲线**：对初学者有一定门槛
- **开发速度**：相比Python开发稍慢
- **生态相对小**：虽然快速发展，但不如Python丰富

### 4.2 项目结构

```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI应用入口
│   ├── core/
│   │   ├── config.py          # 配置管理
│   │   ├── security.py        # 安全认证
│   │   └── database.py        # 数据库连接
│   ├── models/
│   │   ├── user.py            # 用户模型
│   │   ├── level.py           # 关卡模型
│   │   ├── task.py            # 任务模型
│   │   └── report.py          # 报告模型
│   ├── schemas/
│   │   ├── user.py            # 用户数据模式
│   │   ├── level.py           # 关卡数据模式
│   │   └── task.py            # 任务数据模式
│   ├── api/
│   │   ├── auth.py            # 认证接口
│   │   ├── users.py           # 用户接口
│   │   ├── levels.py          # 关卡接口
│   │   ├── tasks.py           # 任务接口
│   │   └── reports.py         # 报告接口
│   └── services/
│       ├── auth_service.py    # 认证服务
│       ├── task_service.py    # 任务服务
│       └── report_service.py  # 报告服务
├── requirements.txt
└── Dockerfile
```

### 4.3 数据模型设计

#### 4.3.1 用户模型
```python
# models/user.py
from sqlalchemy import Column, Integer, String, DateTime, Boolean
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(50), unique=True, index=True)
    email = Column(String(100), unique=True, index=True)
    hashed_password = Column(String(255))
    full_name = Column(String(100))
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime)
    current_level = Column(Integer, default=1)
    organization_id = Column(Integer, nullable=True)  # 后期扩展用
```

#### 4.3.2 关卡模型
```python
# models/level.py
class Level(Base):
    __tablename__ = "levels"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100))  # 例：中心工作
    description = Column(Text)
    order_number = Column(Integer)  # 1-9
    is_active = Column(Boolean, default=True)
    
    # 关联任务
    tasks = relationship("Task", back_populates="level")
```

#### 4.3.3 任务模型
```python
# models/task.py
class Task(Base):
    __tablename__ = "tasks"
    
    id = Column(Integer, primary_key=True)
    level_id = Column(Integer, ForeignKey("levels.id"))
    title = Column(String(200))
    description = Column(Text)
    task_type = Column(String(20))  # text, image, choice
    is_required = Column(Boolean, default=True)
    order_number = Column(Integer)
    
    level = relationship("Level", back_populates="tasks")

class UserTaskProgress(Base):
    __tablename__ = "user_task_progress"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"))
    task_id = Column(Integer, ForeignKey("tasks.id"))
    content = Column(Text)  # 用户填写的内容
    file_urls = Column(JSON)  # 上传的文件URL
    is_completed = Column(Boolean, default=False)
    completed_at = Column(DateTime)
```

### 4.4 API接口设计

#### 4.4.1 认证接口
```python
# api/auth.py
from fastapi import APIRouter, Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

router = APIRouter(prefix="/auth", tags=["authentication"])

@router.post("/login")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """用户登录接口"""
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(status_code=401, detail="用户名或密码错误")
    
    access_token = create_access_token(data={"sub": user.username})
    return {"access_token": access_token, "token_type": "bearer"}

@router.post("/register")
async def register(user_data: UserCreate):
    """用户注册接口"""
    # 检查用户是否已存在
    existing_user = get_user_by_email(user_data.email)
    if existing_user:
        raise HTTPException(status_code=400, detail="邮箱已被注册")
    
    # 创建新用户
    user = create_user(user_data)
    return {"message": "注册成功", "user_id": user.id}
```

#### 4.4.2 关卡接口
```python
# api/levels.py
@router.get("/levels")
async def get_user_levels(current_user: User = Depends(get_current_user)):
    """获取用户可访问的关卡列表"""
    levels = get_levels_for_user(current_user.id)
    return levels

@router.get("/levels/{level_id}")
async def get_level_detail(
    level_id: int, 
    current_user: User = Depends(get_current_user)
):
    """获取关卡详情和任务列表"""
    # 检查用户是否有权限访问该关卡
    if not user_can_access_level(current_user.id, level_id):
        raise HTTPException(status_code=403, detail="关卡未解锁")
    
    level = get_level_with_tasks(level_id, current_user.id)
    return level
```

#### 4.4.3 任务接口
```python
# api/tasks.py
@router.post("/tasks/{task_id}/submit")
async def submit_task(
    task_id: int,
    task_data: TaskSubmission,
    current_user: User = Depends(get_current_user)
):
    """提交任务答案"""
    # 保存任务进度
    progress = save_task_progress(
        user_id=current_user.id,
        task_id=task_id,
        content=task_data.content,
        files=task_data.files
    )
    
    # 检查是否完成了当前关卡所有任务
    if check_level_completed(current_user.id, progress.task.level_id):
        unlock_next_level(current_user.id)
    
    return {"message": "任务提交成功", "progress": progress}
```

## 5. 数据库设计

### 5.1 数据库选择 - PostgreSQL
- **功能强大**：支持JSON字段，适合存储复杂数据
- **性能优秀**：支持大量并发用户
- **扩展性好**：后期可以轻松扩展

### 5.2 数据库表结构

```mermaid
erDiagram
    USERS ||--o{ USER_TASK_PROGRESS : has
    USERS ||--o{ USER_REPORTS : generates
    LEVELS ||--o{ TASKS : contains
    TASKS ||--o{ USER_TASK_PROGRESS : tracks
    
    USERS {
        int id PK
        string username
        string email
        string hashed_password
        string full_name
        boolean is_active
        datetime created_at
        int current_level
        int organization_id
    }
    
    LEVELS {
        int id PK
        string name
        text description
        int order_number
        boolean is_active
    }
    
    TASKS {
        int id PK
        int level_id FK
        string title
        text description
        string task_type
        boolean is_required
        int order_number
    }
    
    USER_TASK_PROGRESS {
        int id PK
        int user_id FK
        int task_id FK
        text content
        json file_urls
        boolean is_completed
        datetime completed_at
    }
    
    USER_REPORTS {
        int id PK
        int user_id FK
        text report_content
        datetime generated_at
    }
```

### 5.3 Redis缓存策略
```python
# 用户会话缓存
user_session_key = f"user_session:{user_id}"
redis.setex(user_session_key, 3600, user_data)  # 1小时过期

# 关卡数据缓存
level_cache_key = f"level_data:{level_id}"
redis.setex(level_cache_key, 1800, level_data)  # 30分钟过期

# 用户进度缓存
progress_key = f"user_progress:{user_id}"
redis.setex(progress_key, 600, progress_data)  # 10分钟过期
```

## 6. 九大关卡任务设计

### 6.1 关卡主题映射

| 关卡序号 | 主题名称 | 核心目标 | 主要任务类型 |
|---------|---------|---------|-------------|
| 1 | 中心工作 | 理解党建与业务融合 | 文本填写、材料上传 |
| 2 | 问题导向 | 识别和分析问题 | 问题列举、案例分析 |
| 3 | 品牌设计 | 品牌形象设计 | 创意设计、图片上传 |
| 4 | 品牌带动 | 品牌影响力扩展 | 方案制定、效果评估 |
| 5 | 载体建设 | 载体平台构建 | 载体规划、资源配置 |
| 6 | 空间呈现 | 物理空间设计 | 空间设计、效果图 |
| 7 | 机制建设 | 制度机制完善 | 制度设计、流程图 |
| 8 | 融合机制 | 深度融合机制 | 融合方案、实施计划 |
| 9 | 守正创新 | 传承与创新 | 创新思路、总结报告 |

### 6.2 任务类型定义

#### 6.2.1 文本填写类任务
```typescript
interface TextTask {
  type: 'text'
  validation: {
    minLength: number
    maxLength: number
    required: boolean
  }
  placeholder: string
  format?: 'plain' | 'rich'  // 纯文本或富文本
}
```

#### 6.2.2 图片上传类任务
```typescript
interface ImageTask {
  type: 'image'
  validation: {
    maxSize: number      // MB
    allowedFormats: string[]  // ['jpg', 'png', 'pdf']
    maxFiles: number
    required: boolean
  }
  description: string
}
```

#### 6.2.3 选择题类任务
```typescript
interface ChoiceTask {
  type: 'choice'
  options: Array<{
    id: string
    text: string
    description?: string
  }>
  multiSelect: boolean
  validation: {
    minSelected: number
    maxSelected: number
  }
}
```

### 6.3 关卡解锁机制

```mermaid
flowchart TD
    A[用户开始关卡] --> B{检查任务完成度}
    B -->|未完成| C[显示剩余任务]
    B -->|全部完成| D[关卡完成]
    D --> E{是否为最后关卡}
    E -->|否| F[解锁下一关卡]
    E -->|是| G[生成学习报告]
    F --> H[发送通知]
    G --> I[发送完成证书]
```

## 7. 文件存储系统

### 7.1 存储策略
```python
# services/file_service.py
class FileService:
    def __init__(self):
        self.base_path = "/app/uploads"
        self.allowed_extensions = ['.jpg', '.jpeg', '.png', '.pdf', '.doc', '.docx']
        self.max_file_size = 10 * 1024 * 1024  # 10MB
    
    async def upload_file(self, file: UploadFile, user_id: int, task_id: int):
        # 验证文件类型和大小
        if not self.validate_file(file):
            raise HTTPException(status_code=400, detail="文件格式不支持")
        
        # 生成唯一文件名
        filename = f"{user_id}_{task_id}_{int(time.time())}_{file.filename}"
        file_path = os.path.join(self.base_path, filename)
        
        # 保存文件
        with open(file_path, "wb") as buffer:
            shutil.copyfileobj(file.file, buffer)
        
        return {
            "filename": filename,
            "url": f"/uploads/{filename}",
            "size": file.size
        }
```

### 7.2 文件组织结构
```
uploads/
├── avatars/          # 用户头像
├── tasks/           # 任务相关文件
│   ├── level_1/     # 按关卡分类
│   ├── level_2/
│   └── ...
└── reports/         # 生成的报告文件
```

## 8. 个人学习报告生成

### 8.1 报告生成流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant F as 前端
    participant B as 后端
    participant R as 报告服务
    participant D as 数据库
    
    U->>F: 完成最后关卡
    F->>B: 请求生成报告
    B->>D: 查询用户所有任务数据
    D-->>B: 返回任务完成情况
    B->>R: 调用报告生成服务
    R->>R: 分析数据、生成报告
    R-->>B: 返回报告内容
    B->>D: 保存报告
    B-->>F: 返回报告URL
    F-->>U: 显示报告预览
```

### 8.2 报告模板设计
```python
# services/report_service.py
class ReportGenerator:
    def generate_user_report(self, user_id: int):
        # 获取用户所有任务完成情况
        user_progress = self.get_user_progress(user_id)
        
        report_data = {
            "user_info": self.get_user_info(user_id),
            "completion_summary": self.analyze_completion(user_progress),
            "level_insights": self.analyze_by_level(user_progress),
            "recommendations": self.generate_recommendations(user_progress),
            "generated_at": datetime.now()
        }
        
        # 使用模板生成HTML报告
        template = self.load_template("user_report.html")
        html_content = template.render(**report_data)
        
        # 生成PDF（可选）
        pdf_path = self.generate_pdf(html_content, user_id)
        
        return {
            "html_content": html_content,
            "pdf_url": pdf_path,
            "report_data": report_data
        }
```

## 9. 用户认证与权限管理

### 9.1 JWT认证流程
```python
# core/security.py
from jose import JWTError, jwt
from datetime import datetime, timedelta

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def verify_token(token: str):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise HTTPException(status_code=401, detail="Token无效")
        return username
    except JWTError:
        raise HTTPException(status_code=401, detail="Token无效")
```

### 9.2 权限控制
```python
# 关卡访问权限检查
def user_can_access_level(user_id: int, level_id: int) -> bool:
    user = get_user(user_id)
    
    # 第一关卡总是可以访问
    if level_id == 1:
        return True
    
    # 检查前一关卡是否完成
    prev_level_completed = check_level_completed(user_id, level_id - 1)
    return prev_level_completed
```

## 10. 开发环境配置

### 10.1 前端开发环境
```bash
# 安装依赖
npm create vue@latest party-sandbox-frontend
cd party-sandbox-frontend
npm install

# 安装额外依赖
npm install element-plus @element-plus/icons-vue
npm install pinia vue-router@4
npm install axios
npm install @types/node --save-dev  # TypeScript支持
```

### 10.2 前端项目配置
```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'path'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src')
    }
  },
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true
      }
    }
  }
})
```

### 10.3 后端开发环境
```bash
# 创建虚拟环境
python -m venv party_sandbox_env
source party_sandbox_env/bin/activate  # Linux/Mac
# party_sandbox_env\Scripts\activate  # Windows

# 安装依赖
pip install fastapi uvicorn sqlalchemy psycopg2-binary
pip install python-multipart python-jose[cryptography]
pip install redis pytest
```

### 10.4 数据库配置
```python
# core/database.py
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "postgresql://username:password@localhost/party_sandbox"

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

## 11. 测试策略

### 11.1 前端测试
```javascript
// tests/components/TaskCard.test.ts
import { mount } from '@vue/test-utils'
import TaskCard from '@/components/TaskCard.vue'

describe('TaskCard', () => {
  test('显示任务标题', () => {
    const wrapper = mount(TaskCard, {
      props: {
        task: {
          title: '测试任务',
          description: '这是一个测试任务',
          type: 'text'
        }
      }
    })
    
    expect(wrapper.text()).toContain('测试任务')
  })
  
  test('文本输入功能', async () => {
    const wrapper = mount(TaskCard, {
      props: {
        task: { type: 'text', title: '文本任务' }
      }
    })
    
    const textarea = wrapper.find('textarea')
    await textarea.setValue('用户输入的内容')
    
    expect(wrapper.emitted('task-updated')).toBeTruthy()
  })
})
```

### 11.2 后端测试
```python
# tests/test_auth.py
import pytest
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_user_registration():
    response = client.post(
        "/auth/register",
        json={
            "username": "testuser",
            "email": "test@example.com", 
            "password": "testpass123",
            "full_name": "测试用户"
        }
    )
    assert response.status_code == 200
    assert "user_id" in response.json()

def test_user_login():
    response = client.post(
        "/auth/login",
        data={"username": "testuser", "password": "testpass123"}
    )
    assert response.status_code == 200
    assert "access_token" in response.json()
```

基于您的需求，我设计了一个完整的党建沙盘Web系统。这个设计采用了前后端分离的架构，使用Vue3+TypeScript构建用户友好的前端界面，Python FastAPI构建高性能的后端服务。

系统的核心特性包括：
- 9个递进式关卡，每个关卡包含多种类型的任务
- 用户认证和权限管理系统
- 文件上传和管理功能
- 自动化的学习报告生成
- 红色金黄色的商务主题设计

请您review一下这个设计方案，看看是否符合您的期望。如果有任何需要调整或补充的地方，请随时告诉我！