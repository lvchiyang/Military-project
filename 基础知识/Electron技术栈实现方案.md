# Electron����ջʵ�ַ���

## 1. Electron�ܹ�����

### 1.1 ElectronӦ�ýṹ

������Ƶ���Ҫ��ϵͳ������Electron����Ӧ�üܹ���ͨ��.exe��װ����������ܹ����£�

```
Electron ����Ӧ��
������ ������ (Main Process)
��   ������ Ӧ���������ڹ���
��   ������ ���ڹ���
��   ������ ϵͳ����
��   ������ ��˷�������
������ ��Ⱦ���� (Renderer Process)
��   ������ Vue.js ǰ��Ӧ��
��   ������ �û�����
��   ������ ҵ���߼�����
������ ��˷��� (Node.js Service)
��   ������ Express API����
��   ������ ҵ���߼�����
��   ������ ���ݿ����
��   ������ �ļ�����
������ ���ݴ洢
    ������ SQLite ���ݿ�
    ������ �����ļ�
    ������ �û��ļ�
```

### 1.2 ����ջѡ�͵���

| �㼶 | ԭ�������� | Electron���� | ����˵�� |
|------|-----------|-------------|----------|
| Ӧ�ÿ�� | WebӦ�� | Electron | ����Ӧ�ÿ�� |
| ǰ�˿�� | Vue 3 | Vue 3 | ���ֲ��� |
| ��˿�� | Spring Boot | Node.js + Express | ����������Electron���ɸ��� |
| ���ݿ� | MySQL | SQLite | Ƕ��ʽ���ݿ⣬������ⰲװ |
| ����ʽ | Docker | .exe��װ�� | ����Ӧ�ð�װ�� |
| ͨ�ŷ�ʽ | HTTP/WebSocket | IPC + HTTP | ���̼�ͨ�� |

## 2. Electron���������

### 2.1 ����������ļ�

**main.js** - ElectronӦ������ڣ�

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
    // Ӧ�����������¼�
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
    
    // Ӧ���˳�ǰ����
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
      title: '�н��ù�ά��ʾ��֤ϵͳ',
      show: false, // ��ʼ���أ�������ɺ���ʾ
      titleBarStyle: 'default'
    });
    
    // ����ǰ��Ӧ��
    if (isDev) {
      this.mainWindow.loadURL('http://localhost:3000');
      this.mainWindow.webContents.openDevTools();
    } else {
      this.mainWindow.loadFile(path.join(__dirname, 'dist/index.html'));
    }
    
    // ���ڼ�����ɺ���ʾ
    this.mainWindow.once('ready-to-show', () => {
      this.mainWindow.show();
      
      // ����ģʽ�´򿪿����߹���
      if (isDev) {
        this.mainWindow.webContents.openDevTools();
      }
    });
    
    // ���ڹر��¼�
    this.mainWindow.on('closed', () => {
      this.mainWindow = null;
    });
  }
  
  async initializeBackend() {
    try {
      // ��ʼ�����ù�����
      const configManager = new ConfigManager();
      await configManager.initialize();
      
      // ��ʼ�����ݿ�
      const dbManager = new DatabaseManager();
      await dbManager.initialize();
      
      // ������˷���
      this.backendService = new BackendService({
        port: 3001,
        database: dbManager,
        config: configManager
      });
      
      await this.backendService.start();
      
      console.log('��˷��������ɹ�');
    } catch (error) {
      console.error('��˷�������ʧ��:', error);
      
      // ��ʾ����Ի���
      dialog.showErrorBox('��������', '��˷�������ʧ�ܣ�����ϵͳ���á�');
      app.quit();
    }
  }
  
  setupIPC() {
    // ��Ŀ�������IPC
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
    
    // Ա���������IPC
    ipcMain.handle('employee:list', async () => {
      return await this.backendService.employeeManager.getAllEmployees();
    });
    
    ipcMain.handle('employee:create', async (event, employeeData) => {
      return await this.backendService.employeeManager.createEmployee(employeeData);
    });
    
    // �ļ��������IPC
    ipcMain.handle('file:upload', async (event, fileData) => {
      return await this.backendService.fileManager.uploadFile(fileData);
    });
    
    ipcMain.handle('file:download', async (event, fileId) => {
      return await this.backendService.fileManager.downloadFile(fileId);
    });
    
    // ϵͳ�������IPC
    ipcMain.handle('system:openPath', async (event, path) => {
      shell.openPath(path);
    });
    
    ipcMain.handle('system:showInFolder', async (event, path) => {
      shell.showItemInFolder(path);
    });
    
    // �Ի������IPC
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

// ����Ӧ��
new MainProcess();
```

### 2.2 Ԥ���ؽű�

**preload.js** - ��ȫ����Ⱦ����API��¶��

```javascript
const { contextBridge, ipcRenderer } = require('electron');

// ͨ��contextBridge��ȫ�ر�¶API����Ⱦ����
contextBridge.exposeInMainWorld('electronAPI', {
  // ��Ŀ����API
  project: {
    create: (projectData) => ipcRenderer.invoke('project:create', projectData),
    list: () => ipcRenderer.invoke('project:list'),
    get: (projectId) => ipcRenderer.invoke('project:get', projectId),
    start: (projectId) => ipcRenderer.invoke('project:start', projectId),
    pause: (projectId) => ipcRenderer.invoke('project:pause', projectId),
    resume: (projectId) => ipcRenderer.invoke('project:resume', projectId),
    delete: (projectId) => ipcRenderer.invoke('project:delete', projectId)
  },
  
  // Ա������API
  employee: {
    list: () => ipcRenderer.invoke('employee:list'),
    create: (employeeData) => ipcRenderer.invoke('employee:create', employeeData),
    update: (employeeId, updates) => ipcRenderer.invoke('employee:update', employeeId, updates),
    delete: (employeeId) => ipcRenderer.invoke('employee:delete', employeeId)
  },
  
  // �ļ�����API
  file: {
    upload: (fileData) => ipcRenderer.invoke('file:upload', fileData),
    download: (fileId) => ipcRenderer.invoke('file:download', fileId),
    delete: (fileId) => ipcRenderer.invoke('file:delete', fileId)
  },
  
  // ϵͳ����API
  system: {
    openPath: (path) => ipcRenderer.invoke('system:openPath', path),
    showInFolder: (path) => ipcRenderer.invoke('system:showInFolder', path),
    getVersion: () => ipcRenderer.invoke('system:getVersion'),
    getPlatform: () => ipcRenderer.invoke('system:getPlatform')
  },
  
  // �Ի���API
  dialog: {
    showOpenDialog: (options) => ipcRenderer.invoke('dialog:showOpenDialog', options),
    showSaveDialog: (options) => ipcRenderer.invoke('dialog:showSaveDialog', options),
    showMessageBox: (options) => ipcRenderer.invoke('dialog:showMessageBox', options)
  },
  
  // �¼�����
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
  
  // �Ƴ��¼�����
  removeAllListeners: (channel) => {
    ipcRenderer.removeAllListeners(channel);
  }
});

// ��¶�汾��Ϣ
contextBridge.exposeInMainWorld('appInfo', {
  version: process.env.npm_package_version || '1.0.0',
  platform: process.platform,
  arch: process.arch
});
```

## 3. ��˷��񼯳�

### 3.1 ��˷���������

**backend/BackendService.js** - ��˷����������

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
    
    // ��ʼ�����������
    this.projectManager = new ProjectManager();
    this.employeeManager = new EmployeeManager();
    this.fileManager = new FileManager();
    this.metricsCalculator = new MetricsCalculator();
    this.eventHandler = new EventHandler();
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  setupMiddleware() {
    // CORS����
    this.app.use(cors({
      origin: ['http://localhost:3000', 'file://'],
      credentials: true
    }));
    
    // JSON����
    this.app.use(express.json({ limit: '50mb' }));
    this.app.use(express.urlencoded({ extended: true, limit: '50mb' }));
    
    // ��̬�ļ�����
    this.app.use('/uploads', express.static(path.join(process.cwd(), 'uploads')));
    this.app.use('/exports', express.static(path.join(process.cwd(), 'exports')));
    
    // ������־
    this.app.use((req, res, next) => {
      console.log(`${req.method} ${req.url}`);
      next();
    });
  }
  
  setupRoutes() {
    // API·��
    this.app.use('/api/projects', require('./routes/projects'));
    this.app.use('/api/employees', require('./routes/employees'));
    this.app.use('/api/files', require('./routes/files'));
    this.app.use('/api/metrics', require('./routes/metrics'));
    this.app.use('/api/tasks', require('./routes/tasks'));
    
    // �������
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
    
    // ������
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
      
      // ����WebSocket
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
          console.log(`��˷��������ڶ˿� ${this.options.port}`);
          resolve();
        }
      });
    });
  }
  
  setupWebSocket() {
    this.io.on('connection', (socket) => {
      console.log(`WebSocket�ͻ�������: ${socket.id}`);
      
      // ������Ŀ����
      socket.on('joinProject', (projectId) => {
        socket.join(`project_${projectId}`);
      });
      
      // �뿪��Ŀ����
      socket.on('leaveProject', (projectId) => {
        socket.leave(`project_${projectId}`);
      });
      
      socket.on('disconnect', () => {
        console.log(`WebSocket�ͻ��˶Ͽ�: ${socket.id}`);
      });
    });
    
    // �����¼�������
    this.eventHandler.setWebSocket(this.io);
  }
  
  stop() {
    if (this.server) {
      this.server.close();
      console.log('��˷�����ֹͣ');
    }
  }
  
  // ��ȡ����ʵ���ķ���
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

### 3.2 ���ݿ������

**backend/DatabaseManager.js** - SQLite���ݿ����

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
    // ȷ������Ŀ¼����
    const dataDir = path.dirname(this.dbPath);
    if (!fs.existsSync(dataDir)) {
      fs.mkdirSync(dataDir, { recursive: true });
    }
    
    // �����ݿ�����
    this.db = await open({
      filename: this.dbPath,
      driver: sqlite3.Database
    });
    
    // �������Լ��
    await this.db.exec('PRAGMA foreign_keys = ON');
    
    // ������ṹ
    await this.createTables();
    
    // �����ʼ����
    await this.insertInitialData();
    
    console.log('���ݿ��ʼ�����');
  }
  
  async createTables() {
    // ��Ŀ��
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
    
    // Ա����
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
    
    // �����
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
    
    // ָ���
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
    
    // �¼���
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
    
    // �ļ���
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
    
    // ��̱���
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
    
    // ��������
    await this.db.exec('CREATE INDEX IF NOT EXISTS idx_projects_status ON projects(status)');
    await this.db.exec('CREATE INDEX IF NOT EXISTS idx_tasks_project_id ON tasks(project_id)');
    await this.db.exec('CREATE INDEX IF NOT EXISTS idx_metrics_project_id ON metrics(project_id)');
    await this.db.exec('CREATE INDEX IF NOT EXISTS idx_events_project_id ON events(project_id)');
  }
  
  async insertInitialData() {
    // ����Ƿ����г�ʼ����
    const employeeCount = await this.db.get('SELECT COUNT(*) as count FROM employees');
    if (employeeCount.count > 0) {
      return; // �������ݣ�������ʼ��
    }
    
    // ����Ĭ��Ա��
    const defaultEmployees = [
      {
        id: 'emp_001',
        name: '����',
        role: 'research',
        dailySalary: 800,
        efficiency: 1.2,
        skills: JSON.stringify(['�������', 'ϵͳ���']),
        experience: 5
      },
      {
        id: 'emp_002',
        name: '����',
        role: 'build',
        dailySalary: 900,
        efficiency: 1.1,
        skills: JSON.stringify(['ǰ�˿���', 'Vue.js']),
        experience: 3
      },
      {
        id: 'emp_003',
        name: '����',
        role: 'build',
        dailySalary: 850,
        efficiency: 1.0,
        skills: JSON.stringify(['��˿���', 'Node.js']),
        experience: 4
      },
      {
        id: 'emp_004',
        name: '����',
        role: 'use',
        dailySalary: 600,
        efficiency: 0.9,
        skills: JSON.stringify(['ϵͳ����', '�û�����']),
        experience: 2
      },
      {
        id: 'emp_005',
        name: '����',
        role: 'manage',
        dailySalary: 1000,
        efficiency: 1.3,
        skills: JSON.stringify(['��Ŀ����', '�Ŷ�Э��']),
        experience: 8
      },
      {
        id: 'emp_006',
        name: '�ܰ�',
        role: 'maintain',
        dailySalary: 750,
        efficiency: 1.0,
        skills: JSON.stringify(['ϵͳά��', '�������']),
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
    
    console.log('��ʼ���ݲ������');
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

## 4. ǰ��VueӦ�ü���

### 4.1 Electron API����

**src/services/electronApi.js** - ��װElectron API���ã�

```javascript
class ElectronApiService {
  constructor() {
    this.isElectron = window.electronAPI !== undefined;
  }
  
  // ��Ŀ����API
  async createProject(projectData) {
    if (this.isElectron) {
      return await window.electronAPI.project.create(projectData);
    } else {
      // ����ģʽ�µ�HTTP API����
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
  
  // Ա������API
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
  
  // �ļ�����API
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
  
  // ϵͳ����API
  async openPath(path) {
    if (this.isElectron) {
      return await window.electronAPI.system.openPath(path);
    } else {
      console.warn('openPath ����Electron�����¿���');
    }
  }
  
  async showInFolder(path) {
    if (this.isElectron) {
      return await window.electronAPI.system.showInFolder(path);
    } else {
      console.warn('showInFolder ����Electron�����¿���');
    }
  }
  
  // �Ի���API
  async showOpenDialog(options) {
    if (this.isElectron) {
      return await window.electronAPI.dialog.showOpenDialog(options);
    } else {
      // Web������ʹ��HTML5�ļ�ѡ��
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
  
  // �¼�����
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
  
  // ��ȡӦ����Ϣ
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

### 4.2 VueӦ�����õ���

**src/main.js** - VueӦ����ڵ�����

```javascript
import { createApp } from 'vue';
import App from './App.vue';
import router from './router';
import { createPinia } from 'pinia';
import ElementPlus from 'element-plus';
import 'element-plus/dist/index.css';
import * as ElementPlusIconsVue from '@element-plus/icons-vue';

// Electron�������
const isElectron = window.electronAPI !== undefined;

const app = createApp(App);

// ע��Element Plusͼ��
for (const [key, component] of Object.entries(ElementPlusIconsVue)) {
  app.component(key, component);
}

// ȫ������
app.config.globalProperties.$isElectron = isElectron;
app.config.globalProperties.$electronAPI = window.electronAPI;

// ���ע��
app.use(createPinia());
app.use(router);
app.use(ElementPlus);

// Electron�ض�����
if (isElectron) {
  // �����Ҽ��˵�����ѡ��
  document.addEventListener('contextmenu', (e) => {
    if (process.env.NODE_ENV === 'production') {
      e.preventDefault();
    }
  });
  
  // ����F12�����߹��ߣ���ѡ��
  document.addEventListener('keydown', (e) => {
    if (process.env.NODE_ENV === 'production' && e.key === 'F12') {
      e.preventDefault();
    }
  });
}

app.mount('#app');
```

### 4.3 ·������

**src/router/index.js** - ·�����ã�

```javascript
import { createRouter, createWebHashHistory } from 'vue-router';

// ʹ��hashģʽ�Լ���Electron file://Э��
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

## 5. ����ͷַ�

### 5.1 Electron Builder����

**package.json** - ��Ŀ���ú͹����ű���

```json
{
  "name": "research-build-use-manage-maintain-system",
  "version": "1.0.0",
  "description": "�н��ù�ά��ʾ��֤ϵͳ",
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
    "productName": "�н��ù�ά��ʾ��֤ϵͳ",
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
      "shortcutName": "�н��ù�άϵͳ"
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

### 5.2 �����ű�

**scripts/build.js** - �Զ��幹���ű���

```javascript
const { execSync } = require('child_process');
const fs = require('fs');
const path = require('path');

console.log('��ʼ�����н��ù�ά��ʾ��֤ϵͳ...');

try {
  // 1. ����ɵĹ����ļ�
  console.log('������Ŀ¼...');
  if (fs.existsSync('dist')) {
    fs.rmSync('dist', { recursive: true, force: true });
  }
  if (fs.existsSync('release')) {
    fs.rmSync('release', { recursive: true, force: true });
  }
  
  // 2. ����Vueǰ��Ӧ��
  console.log('����ǰ��Ӧ��...');
  execSync('npm run build:vue', { stdio: 'inherit' });
  
  // 3. ���ƺ���ļ�
  console.log('׼������ļ�...');
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
  
  // 4. ���ɰ汾��Ϣ
  console.log('���ɰ汾��Ϣ...');
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
  
  // 5. ����ElectronӦ��
  console.log('����ElectronӦ��...');
  execSync('npm run build:electron', { stdio: 'inherit' });
  
  console.log('������ɣ�');
  console.log('��װ��λ��: release/');
  
} catch (error) {
  console.error('����ʧ��:', error.message);
  process.exit(1);
}
```

## 6. ���úͲ���

### 6.1 Ӧ������

**config/app.json** - Ӧ�������ļ���

```json
{
  "app": {
    "name": "�н��ù�ά��ʾ��֤ϵͳ",
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

### 6.2 ��װ��������

**installer.nsh** - NSIS��װ�����Զ���ű���

```nsis
; �Զ��尲װ����ű�
!macro customInstall
  ; ��������Ŀ¼
  CreateDirectory "$APPDATA\${PRODUCT_NAME}\data"
  CreateDirectory "$APPDATA\${PRODUCT_NAME}\logs"
  CreateDirectory "$APPDATA\${PRODUCT_NAME}\uploads"
  CreateDirectory "$APPDATA\${PRODUCT_NAME}\exports"
  
  ; ���������ļ�
  SetOutPath "$APPDATA\${PRODUCT_NAME}"
  File /r "${BUILD_RESOURCES_DIR}\config"
  File /r "${BUILD_RESOURCES_DIR}\templates"
  
  ; �����ļ�Ȩ��
  AccessControl::GrantOnFile "$APPDATA\${PRODUCT_NAME}" "(BU)" "FullAccess"
!macroend

!macro customUnInstall
  ; �����û����ݣ���ѡ��
  MessageBox MB_YESNO "�Ƿ�ɾ��������Ŀ���ݺ������ļ���" IDNO skip_data_removal
  RMDir /r "$APPDATA\${PRODUCT_NAME}"
  skip_data_removal:
!macroend
```

## 7. �����͵���

### 7.1 ������������

**����ģʽ�����ű�**��

```bash
# ��װ����
npm install

# ��������ģʽ��ͬʱ����Vue������������Electron��
npm run dev

# ������Vue����������
npm run dev:vue

# ������Electron����ҪVue�������������У�
npm run dev:electron
```

### 7.2 ��������

**.vscode/launch.json** - VS Code�������ã�

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

## 8. �����Ż�

### 8.1 �����Ż�

- **������ģ��**���������ҵ��ģ��
- **Ԥ����ģ��**��Ԥ����Vueģ����������ٶ�
- **��Դѹ��**��ѹ����̬��Դ���ٰ����
- **�������**������ʹ�û��������Ӧ�ٶ�

### 8.2 �ڴ��Ż�

- **�����**�����ö������GCѹ��
- **�¼���������**����ʱ�����¼�������
- **���ݿ����ӳ�**������������ݿ�����
- **���ļ�����**����ʽ������ļ�

## 9. ��ȫ����

### 9.1 Electron��ȫ

- **����Node����**������Ⱦ�����н���Node.js����
- **���������ĸ���**��ʹ��contextBridge��ȫ�ر�¶API
- **CSP����**���������ݰ�ȫ����
- **Ȩ�޿���**����СȨ��ԭ��

### 9.2 ���ݰ�ȫ

- **���ݿ����**��SQLite���ݿ���ܴ洢
- **�ļ�Ȩ��**���ϸ�����ļ�����Ȩ��
- **������֤**�������û����������֤
- **��־����**��������Ϣ����¼����־

## 10. �ܽ�

ͨ��Electron����ջʵ�ַ�����ϵͳ���߱������ص㣺

1. **����Ӧ������**��ԭ������Ӧ�ý���ͽ���
2. **�򻯲���**����һ.exe��װ���������������
3. **��������**���������ݴ洢��������������
4. **��ƽ̨֧��**��֧��Windows��macOS��Linux
5. **�߶ȼ���**��ǰ���һ�廯�����ٲ����Ӷ�
6. **�û��Ѻ�**����Ϥ������Ӧ�ò���ϰ��

�ü���ջ������ȫ������Ƶ����ĵ���Ҫ��Ϊ�û��ṩ������������Ӧ�ý��������
