# Electron技术栈实现方案

## 1. Electron架构概述

### 1.1 Electron应用结构

根据设计调整要求，系统将采用Electron桌面应用架构，通过.exe安装包部署。整体架构如下：

```
Electron 桌面应用
├── 主进程 (Main Process)
│   ├── 应用生命周期管理
│   ├── 窗口管理
│   ├── 系统集成
│   └── 后端服务启动
├── 渲染进程 (Renderer Process)
│   ├── Vue.js 前端应用
│   ├── 用户界面
│   └── 业务逻辑处理
├── 后端服务 (Node.js Service)
│   ├── Express API服务
│   ├── 业务逻辑处理
│   ├── 数据库操作
│   └── 文件管理
└── 数据存储
    ├── SQLite 数据库
    ├── 配置文件
    └── 用户文件
```

### 1.2 技术栈选型调整

| 层级 | 原技术方案 | Electron方案 | 调整说明 |
|------|-----------|-------------|----------|
| 应用框架 | Web应用 | Electron | 桌面应用框架 |
| 前端框架 | Vue 3 | Vue 3 | 保持不变 |
| 后端框架 | Spring Boot | Node.js + Express | 轻量化，与Electron集成更好 |
| 数据库 | MySQL | SQLite | 嵌入式数据库，无需额外安装 |
| 部署方式 | Docker | .exe安装包 | 桌面应用安装包 |
| 通信方式 | HTTP/WebSocket | IPC + HTTP | 进程间通信 |

## 2. Electron主进程设计

### 2.1 主进程入口文件

**main.js** - Electron应用主入口：

```javascript
const { app, BrowserWindow, ipcMain, dialog, shell } = require('electron');
const path = require('path');
const isDev = process.env.NODE_ENV === 'development';
const { BackendService } = require('./backend/BackendService');
const { DatabaseManager } = require('./backend/DatabaseManager');
const { ConfigManager } = require('./backend/ConfigManager');

class MainProcess {
  constructor() {
    this.mainWindow = null;
    this.backendService = null;
    this.isReady = false;
    
    this.initializeApp();
  }
  
  initializeApp() {
    // 应用生命周期事件
    app.whenReady().then(() => {
      this.createMainWindow();
      this.initializeBackend();
      this.setupIPC();
      this.isReady = true;
    });
    
    app.on('window-all-closed', () => {
      if (process.platform !== 'darwin') {
        this.cleanup();
        app.quit();
      }
    });
    
    app.on('activate', () => {
      if (BrowserWindow.getAllWindows().length === 0) {
        this.createMainWindow();
      }
    });
    
    // 应用退出前清理
    app.on('before-quit', () => {
      this.cleanup();
    });
  }
  
  createMainWindow() {
    this.mainWindow = new BrowserWindow({
      width: 1400,
      height: 900,
      minWidth: 1200,
      minHeight: 800,
      webPreferences: {
        nodeIntegration: false,
        contextIsolation: true,
        enableRemoteModule: false,
        preload: path.join(__dirname, 'preload.js')
      },
      icon: path.join(__dirname, 'assets/icon.png'),
      title: '研建用管维演示验证系统',
      show: false, // 初始隐藏，加载完成后显示
      titleBarStyle: 'default'
    });
    
    // 加载前端应用
    if (isDev) {
      this.mainWindow.loadURL('http://localhost:3000');
      this.mainWindow.webContents.openDevTools();
    } else {
      this.mainWindow.loadFile(path.join(__dirname, 'dist/index.html'));
    }
    
    // 窗口加载完成后显示
    this.mainWindow.once('ready-to-show', () => {
      this.mainWindow.show();
      
      // 开发模式下打开开发者工具
      if (isDev) {
        this.mainWindow.webContents.openDevTools();
      }
    });
    
    // 窗口关闭事件
    this.mainWindow.on('closed', () => {
      this.mainWindow = null;
    });
  }
  
  async initializeBackend() {
    try {
      // 初始化配置管理器
      const configManager = new ConfigManager();
      await configManager.initialize();
      
      // 初始化数据库
      const dbManager = new DatabaseManager();
      await dbManager.initialize();
      
      // 启动后端服务
      this.backendService = new BackendService({
        port: 3001,
        database: dbManager,
        config: configManager
      });
      
      await this.backendService.start();
      
      console.log('后端服务启动成功');
    } catch (error) {
      console.error('后端服务启动失败:', error);
      
      // 显示错误对话框
      dialog.showErrorBox('启动错误', '后端服务启动失败，请检查系统配置。');
      app.quit();
    }
  }
  
  setupIPC() {
    // 项目管理相关IPC
    ipcMain.handle('project:create', async (event, projectData) => {
      return await this.backendService.projectManager.createProject(projectData);
    });
    
    ipcMain.handle('project:list', async () => {
      return await this.backendService.projectManager.getAllProjects();
    });
    
    ipcMain.handle('project:get', async (event, projectId) => {
      return await this.backendService.projectManager.getProject(projectId);
    });
    
    ipcMain.handle('project:start', async (event, projectId) => {
      return await this.backendService.projectManager.startProject(projectId);
    });
    
    ipcMain.handle('project:pause', async (event, projectId) => {
      return await this.backendService.projectManager.pauseProject(projectId);
    });
    
    // 员工管理相关IPC
    ipcMain.handle('employee:list', async () => {
      return await this.backendService.employeeManager.getAllEmployees();
    });
    
    ipcMain.handle('employee:create', async (event, employeeData) => {
      return await this.backendService.employeeManager.createEmployee(employeeData);
    });
    
    // 文件管理相关IPC
    ipcMain.handle('file:upload', async (event, fileData) => {
      return await this.backendService.fileManager.uploadFile(fileData);
    });
    
    ipcMain.handle('file:download', async (event, fileId) => {
      return await this.backendService.fileManager.downloadFile(fileId);
    });
    
    // 系统操作相关IPC
    ipcMain.handle('system:openPath', async (event, path) => {
      shell.openPath(path);
    });
    
    ipcMain.handle('system:showInFolder', async (event, path) => {
      shell.showItemInFolder(path);
    });
    
    // 对话框相关IPC
    ipcMain.handle('dialog:showOpenDialog', async (event, options) => {
      const result = await dialog.showOpenDialog(this.mainWindow, options);
      return result;
    });
    
    ipcMain.handle('dialog:showSaveDialog', async (event, options) => {
      const result = await dialog.showSaveDialog(this.mainWindow, options);
      return result;
    });
  }
  
  cleanup() {
    if (this.backendService) {
      this.backendService.stop();
    }
  }
}

// 启动应用
new MainProcess();
```

### 2.2 预加载脚本

**preload.js** - 安全的渲染进程API暴露：

```javascript
const { contextBridge, ipcRenderer } = require('electron');

// 通过contextBridge安全地暴露API给渲染进程
contextBridge.exposeInMainWorld('electronAPI', {
  // 项目管理API
  project: {
    create: (projectData) => ipcRenderer.invoke('project:create', projectData),
    list: () => ipcRenderer.invoke('project:list'),
    get: (projectId) => ipcRenderer.invoke('project:get', projectId),
    start: (projectId) => ipcRenderer.invoke('project:start', projectId),
    pause: (projectId) => ipcRenderer.invoke('project:pause', projectId),
    resume: (projectId) => ipcRenderer.invoke('project:resume', projectId),
    delete: (projectId) => ipcRenderer.invoke('project:delete', projectId)
  },
  
  // 员工管理API
  employee: {
    list: () => ipcRenderer.invoke('employee:list'),
    create: (employeeData) => ipcRenderer.invoke('employee:create', employeeData),
    update: (employeeId, updates) => ipcRenderer.invoke('employee:update', employeeId, updates),
    delete: (employeeId) => ipcRenderer.invoke('employee:delete', employeeId)
  },
  
  // 文件管理API
  file: {
    upload: (fileData) => ipcRenderer.invoke('file:upload', fileData),
    download: (fileId) => ipcRenderer.invoke('file:download', fileId),
    delete: (fileId) => ipcRenderer.invoke('file:delete', fileId)
  },
  
  // 系统操作API
  system: {
    openPath: (path) => ipcRenderer.invoke('system:openPath', path),
    showInFolder: (path) => ipcRenderer.invoke('system:showInFolder', path),
    getVersion: () => ipcRenderer.invoke('system:getVersion'),
    getPlatform: () => ipcRenderer.invoke('system:getPlatform')
  },
  
  // 对话框API
  dialog: {
    showOpenDialog: (options) => ipcRenderer.invoke('dialog:showOpenDialog', options),
    showSaveDialog: (options) => ipcRenderer.invoke('dialog:showSaveDialog', options),
    showMessageBox: (options) => ipcRenderer.invoke('dialog:showMessageBox', options)
  },
  
  // 事件监听
  on: (channel, callback) => {
    const validChannels = [
      'project:updated',
      'employee:updated',
      'metric:updated',
      'task:completed',
      'system:notification'
    ];
    
    if (validChannels.includes(channel)) {
      ipcRenderer.on(channel, callback);
    }
  },
  
  // 移除事件监听
  removeAllListeners: (channel) => {
    ipcRenderer.removeAllListeners(channel);
  }
});

// 暴露版本信息
contextBridge.exposeInMainWorld('appInfo', {
  version: process.env.npm_package_version || '1.0.0',
  platform: process.platform,
  arch: process.arch
});
```

## 3. 后端服务集成

### 3.1 后端服务启动器

**backend/BackendService.js** - 后端服务管理器：

```javascript
const express = require('express');
const cors = require('cors');
const http = require('http');
const { Server } = require('socket.io');
const path = require('path');

const { ProjectManager } = require('./services/ProjectManager');
const { EmployeeManager } = require('./services/EmployeeManager');
const { FileManager } = require('./services/FileManager');
const { MetricsCalculator } = require('./services/MetricsCalculator');
const { EventHandler } = require('./services/EventHandler');

class BackendService {
  constructor(options) {
    this.options = options;
    this.app = express();
    this.server = null;
    this.io = null;
    
    // 初始化服务管理器
    this.projectManager = new ProjectManager();
    this.employeeManager = new EmployeeManager();
    this.fileManager = new FileManager();
    this.metricsCalculator = new MetricsCalculator();
    this.eventHandler = new EventHandler();
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  setupMiddleware() {
    // CORS配置
    this.app.use(cors({
      origin: ['http://localhost:3000', 'file://'],
      credentials: true
    }));
    
    // JSON解析
    this.app.use(express.json({ limit: '50mb' }));
    this.app.use(express.urlencoded({ extended: true, limit: '50mb' }));
    
    // 静态文件服务
    this.app.use('/uploads', express.static(path.join(process.cwd(), 'uploads')));
    this.app.use('/exports', express.static(path.join(process.cwd(), 'exports')));
    
    // 请求日志
    this.app.use((req, res, next) => {
      console.log(`${req.method} ${req.url}`);
      next();
    });
  }
  
  setupRoutes() {
    // API路由
    this.app.use('/api/projects', require('./routes/projects'));
    this.app.use('/api/employees', require('./routes/employees'));
    this.app.use('/api/files', require('./routes/files'));
    this.app.use('/api/metrics', require('./routes/metrics'));
    this.app.use('/api/tasks', require('./routes/tasks'));
    
    // 健康检查
    this.app.get('/health', (req, res) => {
      res.json({
        status: 'ok',
        timestamp: new Date().toISOString(),
        services: {
          database: this.options.database.isConnected(),
          fileSystem: this.fileManager.isHealthy()
        }
      });
    });
    
    // 错误处理
    this.app.use((error, req, res, next) => {
      console.error('API Error:', error);
      res.status(500).json({
        success: false,
        error: error.message,
        stack: process.env.NODE_ENV === 'development' ? error.stack : undefined
      });
    });
  }
  
  async start() {
    return new Promise((resolve, reject) => {
      this.server = http.createServer(this.app);
      
      // 设置WebSocket
      this.io = new Server(this.server, {
        cors: {
          origin: "*",
          methods: ["GET", "POST"]
        }
      });
      
      this.setupWebSocket();
      
      this.server.listen(this.options.port, 'localhost', (error) => {
        if (error) {
          reject(error);
        } else {
          console.log(`后端服务启动在端口 ${this.options.port}`);
          resolve();
        }
      });
    });
  }
  
  setupWebSocket() {
    this.io.on('connection', (socket) => {
      console.log(`WebSocket客户端连接: ${socket.id}`);
      
      // 加入项目房间
      socket.on('joinProject', (projectId) => {
        socket.join(`project_${projectId}`);
      });
      
      // 离开项目房间
      socket.on('leaveProject', (projectId) => {
        socket.leave(`project_${projectId}`);
      });
      
      socket.on('disconnect', () => {
        console.log(`WebSocket客户端断开: ${socket.id}`);
      });
    });
    
    // 设置事件处理器
    this.eventHandler.setWebSocket(this.io);
  }
  
  stop() {
    if (this.server) {
      this.server.close();
      console.log('后端服务已停止');
    }
  }
  
  // 获取服务实例的方法
  getProjectManager() {
    return this.projectManager;
  }
  
  getEmployeeManager() {
    return this.employeeManager;
  }
  
  getFileManager() {
    return this.fileManager;
  }
  
  getMetricsCalculator() {
    return this.metricsCalculator;
  }
}

module.exports = { BackendService };
```

### 3.2 数据库管理器

**backend/DatabaseManager.js** - SQLite数据库管理：

```javascript
const sqlite3 = require('sqlite3').verbose();
const { open } = require('sqlite');
const path = require('path');
const fs = require('fs');

class DatabaseManager {
  constructor() {
    this.db = null;
    this.dbPath = path.join(process.cwd(), 'data', 'app.db');
  }
  
  async initialize() {
    // 确保数据目录存在
    const dataDir = path.dirname(this.dbPath);
    if (!fs.existsSync(dataDir)) {
      fs.mkdirSync(dataDir, { recursive: true });
    }
    
    // 打开数据库连接
    this.db = await open({
      filename: this.dbPath,
      driver: sqlite3.Database
    });
    
    // 启用外键约束
    await this.db.exec('PRAGMA foreign_keys = ON');
    
    // 创建表结构
    await this.createTables();
    
    // 插入初始数据
    await this.insertInitialData();
    
    console.log('数据库初始化完成');
  }
  
  async createTables() {
    // 项目表
    await this.db.exec(`
      CREATE TABLE IF NOT EXISTS projects (
        id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        description TEXT,
        status TEXT NOT NULL DEFAULT 'pending',
        current_phase TEXT NOT NULL DEFAULT 'research',
        progress REAL DEFAULT 0,
        budget_total REAL NOT NULL,
        budget_used REAL DEFAULT 0,
        time_multiplier REAL DEFAULT 1.0,
        config TEXT,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
      )
    `);
    
    // 员工表
    await this.db.exec(`
      CREATE TABLE IF NOT EXISTS employees (
        id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        role TEXT NOT NULL,
        status TEXT DEFAULT 'idle',
        efficiency REAL DEFAULT 1.0,
        daily_salary REAL NOT NULL,
        total_score INTEGER DEFAULT 0,
        skills TEXT,
        experience INTEGER DEFAULT 0,
        avatar TEXT,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
      )
    `);
    
    // 任务表
    await this.db.exec(`
      CREATE TABLE IF NOT EXISTS tasks (
        id TEXT PRIMARY KEY,
        project_id TEXT NOT NULL,
        name TEXT NOT NULL,
        phase TEXT NOT NULL,
        type TEXT NOT NULL,
        duration INTEGER NOT NULL,
        required_role TEXT NOT NULL,
        assigned_employee_id TEXT,
        status TEXT DEFAULT 'pending',
        progress REAL DEFAULT 0,
        start_time DATETIME,
        end_time DATETIME,
        dependencies TEXT,
        config TEXT,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY (project_id) REFERENCES projects(id),
        FOREIGN KEY (assigned_employee_id) REFERENCES employees(id)
      )
    `);
    
    // 指标表
    await this.db.exec(`
      CREATE TABLE IF NOT EXISTS metrics (
        id TEXT PRIMARY KEY,
        project_id TEXT NOT NULL,
        metric_id TEXT NOT NULL,
        metric_name TEXT NOT NULL,
        category TEXT NOT NULL,
        value TEXT NOT NULL,
        unit TEXT,
        trend TEXT,
        change_percent REAL DEFAULT 0,
        recorded_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY (project_id) REFERENCES projects(id)
      )
    `);
    
    // 事件表
    await this.db.exec(`
      CREATE TABLE IF NOT EXISTS events (
        id TEXT PRIMARY KEY,
        type TEXT NOT NULL,
        project_id TEXT,
        task_id TEXT,
        employee_id TEXT,
        timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
        data TEXT,
        source TEXT,
        metadata TEXT,
        FOREIGN KEY (project_id) REFERENCES projects(id),
        FOREIGN KEY (task_id) REFERENCES tasks(id),
        FOREIGN KEY (employee_id) REFERENCES employees(id)
      )
    `);
    
    // 文件表
    await this.db.exec(`
      CREATE TABLE IF NOT EXISTS files (
        id TEXT PRIMARY KEY,
        project_id TEXT,
        original_name TEXT NOT NULL,
        filename TEXT NOT NULL,
        file_path TEXT NOT NULL,
        file_size INTEGER NOT NULL,
        mime_type TEXT NOT NULL,
        category TEXT DEFAULT 'other',
        uploaded_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY (project_id) REFERENCES projects(id)
      )
    `);
    
    // 里程碑表
    await this.db.exec(`
      CREATE TABLE IF NOT EXISTS milestones (
        id TEXT PRIMARY KEY,
        project_id TEXT NOT NULL,
        phase TEXT NOT NULL,
        name TEXT NOT NULL,
        description TEXT,
        deliverables TEXT,
        file_path TEXT,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY (project_id) REFERENCES projects(id)
      )
    `);
    
    // 创建索引
    await this.db.exec('CREATE INDEX IF NOT EXISTS idx_projects_status ON projects(status)');
    await this.db.exec('CREATE INDEX IF NOT EXISTS idx_tasks_project_id ON tasks(project_id)');
    await this.db.exec('CREATE INDEX IF NOT EXISTS idx_metrics_project_id ON metrics(project_id)');
    await this.db.exec('CREATE INDEX IF NOT EXISTS idx_events_project_id ON events(project_id)');
  }
  
  async insertInitialData() {
    // 检查是否已有初始数据
    const employeeCount = await this.db.get('SELECT COUNT(*) as count FROM employees');
    if (employeeCount.count > 0) {
      return; // 已有数据，跳过初始化
    }
    
    // 插入默认员工
    const defaultEmployees = [
      {
        id: 'emp_001',
        name: '张三',
        role: 'research',
        dailySalary: 800,
        efficiency: 1.2,
        skills: JSON.stringify(['需求分析', '系统设计']),
        experience: 5
      },
      {
        id: 'emp_002',
        name: '李四',
        role: 'build',
        dailySalary: 900,
        efficiency: 1.1,
        skills: JSON.stringify(['前端开发', 'Vue.js']),
        experience: 3
      },
      {
        id: 'emp_003',
        name: '王五',
        role: 'build',
        dailySalary: 850,
        efficiency: 1.0,
        skills: JSON.stringify(['后端开发', 'Node.js']),
        experience: 4
      },
      {
        id: 'emp_004',
        name: '赵六',
        role: 'use',
        dailySalary: 600,
        efficiency: 0.9,
        skills: JSON.stringify(['系统测试', '用户体验']),
        experience: 2
      },
      {
        id: 'emp_005',
        name: '孙七',
        role: 'manage',
        dailySalary: 1000,
        efficiency: 1.3,
        skills: JSON.stringify(['项目管理', '团队协调']),
        experience: 8
      },
      {
        id: 'emp_006',
        name: '周八',
        role: 'maintain',
        dailySalary: 750,
        efficiency: 1.0,
        skills: JSON.stringify(['系统维护', '问题诊断']),
        experience: 6
      }
    ];
    
    for (const employee of defaultEmployees) {
      await this.db.run(
        `INSERT INTO employees (id, name, role, daily_salary, efficiency, skills, experience)
         VALUES (?, ?, ?, ?, ?, ?, ?)`,
        [employee.id, employee.name, employee.role, employee.dailySalary, 
         employee.efficiency, employee.skills, employee.experience]
      );
    }
    
    console.log('初始数据插入完成');
  }
  
  getDatabase() {
    return this.db;
  }
  
  isConnected() {
    return this.db !== null;
  }
  
  async close() {
    if (this.db) {
      await this.db.close();
      this.db = null;
    }
  }
}

module.exports = { DatabaseManager };
```

## 4. 前端Vue应用集成

### 4.1 Electron API服务

**src/services/electronApi.js** - 封装Electron API调用：

```javascript
class ElectronApiService {
  constructor() {
    this.isElectron = window.electronAPI !== undefined;
  }
  
  // 项目管理API
  async createProject(projectData) {
    if (this.isElectron) {
      return await window.electronAPI.project.create(projectData);
    } else {
      // 开发模式下的HTTP API调用
      const response = await fetch('/api/projects', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(projectData)
      });
      return await response.json();
    }
  }
  
  async getProjects() {
    if (this.isElectron) {
      return await window.electronAPI.project.list();
    } else {
      const response = await fetch('/api/projects');
      return await response.json();
    }
  }
  
  async getProject(projectId) {
    if (this.isElectron) {
      return await window.electronAPI.project.get(projectId);
    } else {
      const response = await fetch(`/api/projects/${projectId}`);
      return await response.json();
    }
  }
  
  async startProject(projectId) {
    if (this.isElectron) {
      return await window.electronAPI.project.start(projectId);
    } else {
      const response = await fetch(`/api/projects/${projectId}/start`, {
        method: 'POST'
      });
      return await response.json();
    }
  }
  
  // 员工管理API
  async getEmployees() {
    if (this.isElectron) {
      return await window.electronAPI.employee.list();
    } else {
      const response = await fetch('/api/employees');
      return await response.json();
    }
  }
  
  async createEmployee(employeeData) {
    if (this.isElectron) {
      return await window.electronAPI.employee.create(employeeData);
    } else {
      const response = await fetch('/api/employees', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(employeeData)
      });
      return await response.json();
    }
  }
  
  // 文件管理API
  async uploadFile(fileData) {
    if (this.isElectron) {
      return await window.electronAPI.file.upload(fileData);
    } else {
      const formData = new FormData();
      formData.append('file', fileData.file);
      formData.append('projectId', fileData.projectId);
      
      const response = await fetch('/api/files/upload', {
        method: 'POST',
        body: formData
      });
      return await response.json();
    }
  }
  
  // 系统操作API
  async openPath(path) {
    if (this.isElectron) {
      return await window.electronAPI.system.openPath(path);
    } else {
      console.warn('openPath 仅在Electron环境下可用');
    }
  }
  
  async showInFolder(path) {
    if (this.isElectron) {
      return await window.electronAPI.system.showInFolder(path);
    } else {
      console.warn('showInFolder 仅在Electron环境下可用');
    }
  }
  
  // 对话框API
  async showOpenDialog(options) {
    if (this.isElectron) {
      return await window.electronAPI.dialog.showOpenDialog(options);
    } else {
      // Web环境下使用HTML5文件选择
      return new Promise((resolve) => {
        const input = document.createElement('input');
        input.type = 'file';
        input.multiple = options.properties?.includes('multiSelections');
        input.accept = options.filters?.map(f => f.extensions.map(e => `.${e}`).join(',')).join(',');
        
        input.onchange = (e) => {
          const files = Array.from(e.target.files);
          resolve({
            canceled: false,
            filePaths: files.map(f => f.path || f.name),
            files: files
          });
        };
        
        input.click();
      });
    }
  }
  
  // 事件监听
  onProjectUpdated(callback) {
    if (this.isElectron) {
      window.electronAPI.on('project:updated', callback);
    }
  }
  
  onMetricUpdated(callback) {
    if (this.isElectron) {
      window.electronAPI.on('metric:updated', callback);
    }
  }
  
  // 获取应用信息
  getAppInfo() {
    if (this.isElectron && window.appInfo) {
      return window.appInfo;
    } else {
      return {
        version: '1.0.0',
        platform: 'web',
        arch: 'unknown'
      };
    }
  }
}

export default new ElectronApiService();
```

### 4.2 Vue应用配置调整

**src/main.js** - Vue应用入口调整：

```javascript
import { createApp } from 'vue';
import App from './App.vue';
import router from './router';
import { createPinia } from 'pinia';
import ElementPlus from 'element-plus';
import 'element-plus/dist/index.css';
import * as ElementPlusIconsVue from '@element-plus/icons-vue';

// Electron环境检测
const isElectron = window.electronAPI !== undefined;

const app = createApp(App);

// 注册Element Plus图标
for (const [key, component] of Object.entries(ElementPlusIconsVue)) {
  app.component(key, component);
}

// 全局属性
app.config.globalProperties.$isElectron = isElectron;
app.config.globalProperties.$electronAPI = window.electronAPI;

// 插件注册
app.use(createPinia());
app.use(router);
app.use(ElementPlus);

// Electron特定配置
if (isElectron) {
  // 禁用右键菜单（可选）
  document.addEventListener('contextmenu', (e) => {
    if (process.env.NODE_ENV === 'production') {
      e.preventDefault();
    }
  });
  
  // 禁用F12开发者工具（可选）
  document.addEventListener('keydown', (e) => {
    if (process.env.NODE_ENV === 'production' && e.key === 'F12') {
      e.preventDefault();
    }
  });
}

app.mount('#app');
```

### 4.3 路由配置

**src/router/index.js** - 路由配置：

```javascript
import { createRouter, createWebHashHistory } from 'vue-router';

// 使用hash模式以兼容Electron file://协议
const router = createRouter({
  history: createWebHashHistory(),
  routes: [
    {
      path: '/',
      redirect: '/login'
    },
    {
      path: '/login',
      name: 'Login',
      component: () => import('../views/Login.vue')
    },
    {
      path: '/admin',
      name: 'AdminDashboard',
      component: () => import('../views/AdminDashboard.vue'),
      children: [
        {
          path: 'projects',
          name: 'ProjectManagement',
          component: () => import('../views/ProjectManagement.vue')
        },
        {
          path: 'employees',
          name: 'EmployeeManagement',
          component: () => import('../views/EmployeeManagement.vue')
        },
        {
          path: 'settings',
          name: 'SystemSettings',
          component: () => import('../views/SystemSettings.vue')
        }
      ]
    },
    {
      path: '/employee',
      name: 'EmployeeDashboard',
      component: () => import('../views/EmployeeDashboard.vue')
    },
    {
      path: '/project/:id',
      name: 'ProjectDetail',
      component: () => import('../views/ProjectDetail.vue')
    },
    {
      path: '/monitoring',
      name: 'MonitoringApp',
      component: () => import('../views/MonitoringApp.vue')
    }
  ]
});

export default router;
```

## 5. 打包和分发

### 5.1 Electron Builder配置

**package.json** - 项目配置和构建脚本：

```json
{
  "name": "research-build-use-manage-maintain-system",
  "version": "1.0.0",
  "description": "研建用管维演示验证系统",
  "main": "main.js",
  "scripts": {
    "dev": "concurrently \"npm run dev:vue\" \"npm run dev:electron\"",
    "dev:vue": "vite",
    "dev:electron": "wait-on http://localhost:3000 && electron .",
    "build": "npm run build:vue && npm run build:electron",
    "build:vue": "vite build",
    "build:electron": "electron-builder",
    "build:win": "electron-builder --win",
    "build:mac": "electron-builder --mac",
    "build:linux": "electron-builder --linux",
    "preview": "vite preview",
    "postinstall": "electron-builder install-app-deps"
  },
  "build": {
    "appId": "com.company.research-system",
    "productName": "研建用管维演示验证系统",
    "directories": {
      "output": "release",
      "buildResources": "build"
    },
    "files": [
      "main.js",
      "preload.js",
      "backend/**/*",
      "dist/**/*",
      "node_modules/**/*",
      "package.json"
    ],
    "extraResources": [
      {
        "from": "config",
        "to": "config"
      },
      {
        "from": "templates",
        "to": "templates"
      }
    ],
    "win": {
      "target": [
        {
          "target": "nsis",
          "arch": ["x64"]
        }
      ],
      "icon": "build/icon.ico",
      "requestedExecutionLevel": "asInvoker"
    },
    "nsis": {
      "oneClick": false,
      "allowElevation": true,
      "allowToChangeInstallationDirectory": true,
      "installerIcon": "build/icon.ico",
      "uninstallerIcon": "build/icon.ico",
      "installerHeaderIcon": "build/icon.ico",
      "createDesktopShortcut": true,
      "createStartMenuShortcut": true,
      "shortcutName": "研建用管维系统"
    },
    "mac": {
      "target": "dmg",
      "icon": "build/icon.icns",
      "category": "public.app-category.productivity"
    },
    "linux": {
      "target": "AppImage",
      "icon": "build/icon.png",
      "category": "Office"
    }
  },
  "devDependencies": {
    "electron": "^22.0.0",
    "electron-builder": "^24.0.0",
    "concurrently": "^7.6.0",
    "wait-on": "^7.0.1",
    "vite": "^4.0.0",
    "@vitejs/plugin-vue": "^4.0.0"
  },
  "dependencies": {
    "vue": "^3.2.0",
    "vue-router": "^4.1.0",
    "pinia": "^2.0.0",
    "element-plus": "^2.2.0",
    "@element-plus/icons-vue": "^2.0.0",
    "express": "^4.18.0",
    "cors": "^2.8.5",
    "socket.io": "^4.6.0",
    "sqlite3": "^5.1.0",
    "sqlite": "^4.1.0",
    "multer": "^1.4.5",
    "uuid": "^9.0.0",
    "moment": "^2.29.0",
    "lodash": "^4.17.0",
    "winston": "^3.8.0"
  }
}
```

### 5.2 构建脚本

**scripts/build.js** - 自定义构建脚本：

```javascript
const { execSync } = require('child_process');
const fs = require('fs');
const path = require('path');

console.log('开始构建研建用管维演示验证系统...');

try {
  // 1. 清理旧的构建文件
  console.log('清理构建目录...');
  if (fs.existsSync('dist')) {
    fs.rmSync('dist', { recursive: true, force: true });
  }
  if (fs.existsSync('release')) {
    fs.rmSync('release', { recursive: true, force: true });
  }
  
  // 2. 构建Vue前端应用
  console.log('构建前端应用...');
  execSync('npm run build:vue', { stdio: 'inherit' });
  
  // 3. 复制后端文件
  console.log('准备后端文件...');
  const backendFiles = [
    'backend',
    'config',
    'templates'
  ];
  
  backendFiles.forEach(dir => {
    if (fs.existsSync(dir)) {
      fs.cpSync(dir, path.join('dist', dir), { recursive: true });
    }
  });
  
  // 4. 生成版本信息
  console.log('生成版本信息...');
  const packageJson = JSON.parse(fs.readFileSync('package.json', 'utf8'));
  const versionInfo = {
    version: packageJson.version,
    buildTime: new Date().toISOString(),
    platform: process.platform,
    arch: process.arch
  };
  fs.writeFileSync(
    path.join('dist', 'version.json'),
    JSON.stringify(versionInfo, null, 2)
  );
  
  // 5. 构建Electron应用
  console.log('构建Electron应用...');
  execSync('npm run build:electron', { stdio: 'inherit' });
  
  console.log('构建完成！');
  console.log('安装包位置: release/');
  
} catch (error) {
  console.error('构建失败:', error.message);
  process.exit(1);
}
```

## 6. 配置和部署

### 6.1 应用配置

**config/app.json** - 应用配置文件：

```json
{
  "app": {
    "name": "研建用管维演示验证系统",
    "version": "1.0.0",
    "environment": "production"
  },
  "database": {
    "path": "data/app.db",
    "maxConnections": 10,
    "timeout": 30000
  },
  "server": {
    "port": 3001,
    "host": "localhost"
  },
  "files": {
    "uploadPath": "uploads",
    "exportPath": "exports",
    "maxFileSize": "50MB",
    "allowedTypes": [".doc", ".docx", ".pdf", ".xls", ".xlsx", ".jpg", ".jpeg", ".png", ".tiff"]
  },
  "metrics": {
    "updateInterval": 5000,
    "alertThreshold": 0.8,
    "historyRetention": 30
  },
  "ui": {
    "theme": "default",
    "language": "zh-CN",
    "autoSave": true,
    "notifications": true
  }
}
```

### 6.2 安装程序配置

**installer.nsh** - NSIS安装程序自定义脚本：

```nsis
; 自定义安装程序脚本
!macro customInstall
  ; 创建数据目录
  CreateDirectory "$APPDATA\${PRODUCT_NAME}\data"
  CreateDirectory "$APPDATA\${PRODUCT_NAME}\logs"
  CreateDirectory "$APPDATA\${PRODUCT_NAME}\uploads"
  CreateDirectory "$APPDATA\${PRODUCT_NAME}\exports"
  
  ; 复制配置文件
  SetOutPath "$APPDATA\${PRODUCT_NAME}"
  File /r "${BUILD_RESOURCES_DIR}\config"
  File /r "${BUILD_RESOURCES_DIR}\templates"
  
  ; 设置文件权限
  AccessControl::GrantOnFile "$APPDATA\${PRODUCT_NAME}" "(BU)" "FullAccess"
!macroend

!macro customUnInstall
  ; 清理用户数据（可选）
  MessageBox MB_YESNO "是否删除所有项目数据和配置文件？" IDNO skip_data_removal
  RMDir /r "$APPDATA\${PRODUCT_NAME}"
  skip_data_removal:
!macroend
```

## 7. 开发和调试

### 7.1 开发环境设置

**开发模式启动脚本**：

```bash
# 安装依赖
npm install

# 启动开发模式（同时启动Vue开发服务器和Electron）
npm run dev

# 仅启动Vue开发服务器
npm run dev:vue

# 仅启动Electron（需要Vue开发服务器运行）
npm run dev:electron
```

### 7.2 调试配置

**.vscode/launch.json** - VS Code调试配置：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Electron Main",
      "type": "node",
      "request": "launch",
      "cwd": "${workspaceFolder}",
      "program": "${workspaceFolder}/main.js",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
      "runtimeArgs": [
        "--remote-debugging-port=9223"
      ],
      "env": {
        "NODE_ENV": "development"
      }
    },
    {
      "name": "Debug Electron Renderer",
      "type": "chrome",
      "request": "attach",
      "port": 9223,
      "webRoot": "${workspaceFolder}/src",
      "timeout": 30000
    }
  ]
}
```

## 8. 性能优化

### 8.1 启动优化

- **懒加载模块**：按需加载业务模块
- **预编译模板**：预编译Vue模板提高启动速度
- **资源压缩**：压缩静态资源减少包体积
- **缓存策略**：合理使用缓存提高响应速度

### 8.2 内存优化

- **对象池**：重用对象减少GC压力
- **事件监听清理**：及时清理事件监听器
- **数据库连接池**：合理管理数据库连接
- **大文件处理**：流式处理大文件

## 9. 安全考虑

### 9.1 Electron安全

- **禁用Node集成**：在渲染进程中禁用Node.js集成
- **启用上下文隔离**：使用contextBridge安全地暴露API
- **CSP策略**：设置内容安全策略
- **权限控制**：最小权限原则

### 9.2 数据安全

- **数据库加密**：SQLite数据库加密存储
- **文件权限**：严格控制文件访问权限
- **输入验证**：所有用户输入进行验证
- **日志脱敏**：敏感信息不记录到日志

## 10. 总结

通过Electron技术栈实现方案，系统将具备以下特点：

1. **桌面应用体验**：原生桌面应用界面和交互
2. **简化部署**：单一.exe安装包，无需额外依赖
3. **离线运行**：本地数据存储，无需网络连接
4. **跨平台支持**：支持Windows、macOS、Linux
5. **高度集成**：前后端一体化，减少部署复杂度
6. **用户友好**：熟悉的桌面应用操作习惯

该技术栈调整完全满足设计调整文档的要求，为用户提供了完整的桌面应用解决方案。
