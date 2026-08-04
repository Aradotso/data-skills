---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and intelligent ticket routing
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics for user risk detection
  - build a user management dashboard with task tracking
  - implement JWT authentication for admin and user roles
  - create AI-powered ticket classification system
  - add burnout detection to user management app
  - set up kanban board with time tracking
  - configure ML service for anomaly detection
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack JavaScript application that combines traditional user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control (Admin/User) with JWT authentication
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support System**: Ticket management with AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Dashboards**: Separate admin and user dashboards with real-time insights

The system uses a microservices architecture with a React frontend, Node.js backend, MongoDB database, and FastAPI ML service.

## Installation

### Prerequisites

```bash
# Ensure you have installed:
# - Node.js (v14+)
# - Python (v3.8+)
# - MongoDB (running instance)
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt
```

### Environment Configuration

Create `.env` files for each service:

**backend/.env**:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**frontend/.env**:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

**ml-service/.env**:
```env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the Application

### Start Backend (Port 5000)

```bash
cd backend
npm start
```

### Start ML Service (Port 8000)

```bash
cd ml-service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Start Frontend (Port 3000)

```bash
cd frontend
npm start
```

## Key Backend API Patterns

### Authentication

```javascript
// backend/controllers/authController.js
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register new user
exports.register = async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ success: true, token, user });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};

// Login
exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ success: false, message: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ success: false, message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ success: true, token, user });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

exports.protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ success: false, message: 'Not authorized' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ success: false, message: 'Token invalid' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        success: false, 
        message: `Role ${req.user.role} is not authorized` 
      });
    }
    next();
  };
};
```

### Task Management

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

// Create task (Admin only)
exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: 'todo',
      dueDate,
      createdBy: req.user.id
    });
    
    await task.save();
    res.status(201).json({ success: true, task });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};

// Get user tasks
exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, tasks });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};

// Update task status
exports.updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ success: false, message: 'Task not found' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json({ success: true, task });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};
```

### Support Ticket System

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

// Create support ticket with AI classification
exports.createTicket = async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    let category = 'general';
    let aiPriority = priority;
    
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      category = mlResponse.data.category;
      aiPriority = mlResponse.data.suggested_priority;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      priority: aiPriority,
      category,
      createdBy: req.user.id,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json({ success: true, ticket });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};

// Get all tickets (Admin)
exports.getAllTickets = async (req, res) => {
  try {
    const tickets = await Ticket.find()
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, tickets });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};
```

## Frontend Patterns

### API Service Setup

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

// Create axios instance
const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Auth services
export const authService = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (userData) => api.post('/auth/register', userData),
  getCurrentUser: () => api.get('/auth/me')
};

// Task services
export const taskService = {
  getTasks: () => api.get('/tasks/my-tasks'),
  createTask: (taskData) => api.post('/tasks', taskData),
  updateTask: (id, updates) => api.put(`/tasks/${id}`, updates),
  updateStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  deleteTask: (id) => api.delete(`/tasks/${id}`)
};

// Ticket services
export const ticketService = {
  createTicket: (ticketData) => api.post('/tickets', ticketData),
  getMyTickets: () => api.get('/tickets/my-tickets'),
  getAllTickets: () => api.get('/tickets'),
  updateTicket: (id, updates) => api.put(`/tickets/${id}`, updates)
};

export default api;
```

### Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import { authService } from '../services/api';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    checkAuth();
  }, []);

  const checkAuth = async () => {
    const token = localStorage.getItem('token');
    if (token) {
      try {
        const response = await authService.getCurrentUser();
        setUser(response.data.user);
      } catch (error) {
        localStorage.removeItem('token');
      }
    }
    setLoading(false);
  };

  const login = async (credentials) => {
    const response = await authService.login(credentials);
    localStorage.setItem('token', response.data.token);
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskService.getTasks();
      const categorized = {
        todo: response.data.tasks.filter(t => t.status === 'todo'),
        inProgress: response.data.tasks.filter(t => t.status === 'in-progress'),
        done: response.data.tasks.filter(t => t.status === 'done')
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const handleDrop = async (taskId, newStatus) => {
    try {
      await taskService.updateStatus(taskId, newStatus);
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const Column = ({ title, status, tasks }) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      <div 
        className="task-list"
        onDrop={(e) => {
          e.preventDefault();
          const taskId = e.dataTransfer.getData('taskId');
          handleDrop(taskId, status);
        }}
        onDragOver={(e) => e.preventDefault()}
      >
        {tasks.map(task => (
          <div
            key={task._id}
            className="task-card"
            draggable
            onDragStart={(e) => e.dataTransfer.setData('taskId', task._id)}
          >
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority ${task.priority}`}>{task.priority}</span>
          </div>
        ))}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <Column title="To Do" status="todo" tasks={tasks.todo} />
      <Column title="In Progress" status="in-progress" tasks={tasks.inProgress} />
      <Column title="Done" status="done" tasks={tasks.done} />
    </div>
  );
};

export default KanbanBoard;
```

## ML Service API

### FastAPI Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Dict
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models (load or initialize)
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: int
    failed_logins: int
    data_access_volume: int
    after_hours_activity: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_task_duration: float
    overtime_hours: float

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """
    Classify support ticket into categories
    """
    # Simple rule-based classification (replace with trained model)
    text = (request.title + " " + request.description).lower()
    
    if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
        category = 'technical'
        priority = 'high'
    elif any(word in text for word in ['password', 'login', 'access', 'permission']):
        category = 'access'
        priority = 'high'
    elif any(word in text for word in ['feature', 'request', 'enhancement']):
        category = 'feature_request'
        priority = 'low'
    else:
        category = 'general'
        priority = 'medium'
    
    return {
        "category": category,
        "suggested_priority": priority,
        "confidence": 0.85
    }

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """
    Predict user risk score based on behavior
    """
    # Calculate risk score (0-100)
    risk_score = 0
    factors = []
    
    # Failed login attempts
    if request.failed_logins > 5:
        risk_score += 30
        factors.append("High failed login attempts")
    
    # Unusual data access
    if request.data_access_volume > 1000:
        risk_score += 25
        factors.append("Unusually high data access")
    
    # After hours activity
    if request.after_hours_activity > 50:
        risk_score += 20
        factors.append("Significant after-hours activity")
    
    # Low login frequency (potentially compromised)
    if request.login_frequency < 5:
        risk_score += 15
        factors.append("Irregular login pattern")
    
    risk_level = "low" if risk_score < 30 else "medium" if risk_score < 60 else "high"
    
    return {
        "user_id": request.user_id,
        "risk_score": min(risk_score, 100),
        "risk_level": risk_level,
        "risk_factors": factors,
        "recommendation": "Investigate user activity" if risk_score > 60 else "Monitor"
    }

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """
    Analyze employee burnout risk
    """
    burnout_score = 0
    indicators = []
    
    # Task completion ratio
    completion_ratio = request.tasks_completed / max(request.tasks_assigned, 1)
    if completion_ratio < 0.5:
        burnout_score += 30
        indicators.append("Low task completion rate")
    
    # Average task duration (in hours)
    if request.avg_task_duration > 8:
        burnout_score += 25
        indicators.append("Extended task durations")
    
    # Overtime hours
    if request.overtime_hours > 20:
        burnout_score += 35
        indicators.append("Excessive overtime")
    
    # High workload
    if request.tasks_assigned > 15:
        burnout_score += 20
        indicators.append("High task load")
    
    burnout_level = "low" if burnout_score < 30 else "moderate" if burnout_score < 60 else "high"
    
    return {
        "user_id": request.user_id,
        "burnout_score": min(burnout_score, 100),
        "burnout_level": burnout_level,
        "indicators": indicators,
        "recommendation": "Reduce workload immediately" if burnout_score > 70 else "Monitor workload"
    }

@app.post("/predict-project-delay")
async def predict_project_delay(
    tasks_total: int,
    tasks_completed: int,
    days_elapsed: int,
    days_remaining: int
):
    """
    Predict if project will be delayed
    """
    completion_rate = tasks_completed / max(tasks_total, 1)
    time_elapsed_ratio = days_elapsed / max(days_elapsed + days_remaining, 1)
    
    # If completion rate is behind time elapsed
    delay_probability = 0
    if completion_rate < time_elapsed_ratio:
        delay_probability = min((time_elapsed_ratio - completion_rate) * 100, 95)
    
    will_delay = delay_probability > 50
    
    return {
        "will_delay": will_delay,
        "delay_probability": round(delay_probability, 2),
        "completion_percentage": round(completion_rate * 100, 2),
        "recommendation": "Increase resources" if will_delay else "On track"
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Integrating ML Service in Backend

```javascript
// backend/services/mlService.js
const axios = require('axios');

const ML_API = process.env.ML_SERVICE_URL;

class MLService {
  async classifyTicket(title, description) {
    try {
      const response = await axios.post(`${ML_API}/classify-ticket`, {
        title,
        description
      });
      return response.data;
    } catch (error) {
      console.error('ML classification error:', error.message);
      return { category: 'general', suggested_priority: 'medium' };
    }
  }

  async predictUserRisk(userData) {
    try {
      const response = await axios.post(`${ML_API}/predict-risk`, userData);
      return response.data;
    } catch (error) {
      console.error('Risk prediction error:', error.message);
      return null;
    }
  }

  async analyzeBurnout(userMetrics) {
    try {
      const response = await axios.post(`${ML_API}/analyze-burnout`, userMetrics);
      return response.data;
    } catch (error) {
      console.error('Burnout analysis error:', error.message);
      return null;
    }
  }

  async predictProjectDelay(projectMetrics) {
    try {
      const response = await axios.post(`${ML_API}/predict-project-delay`, null, {
        params: projectMetrics
      });
      return response.data;
    } catch (error) {
      console.error('Project delay prediction error:', error.message);
      return null;
    }
  }
}

module.exports = new MLService();
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['admin', 'user'],
    default: 'user'
  },
  department: String,
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: Date,
  failedLoginAttempts: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: Date,
  completedAt: Date,
  timeTracked: {
    type: Number,
    default: 0 // in minutes
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'access', 'feature_request', 'general'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  resolution: String,
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Workflows

### Complete User Registration and Login Flow

```javascript
// Example usage in frontend
import { authService } from './services/api';

// Registration
async function registerUser() {
  try {
    const userData = {
      username: 'johndoe',
      email: 'john@example.com',
      password: 'SecurePass123!',
      role: 'user',
      department: 'Engineering'
    };
    
    const response = await authService.register(userData);
    localStorage.setItem('token', response.data.token);
    console.log('Registered:', response.data.user);
  } catch (error) {
    console.error('Registration failed:', error.response.data);
  }
}

// Login
async function loginUser() {
  try {
    const credentials = {
      email: 'john@example.com',
      password: 'SecurePass123!'
    };
    
    const response = await authService.login(credentials);
    localStorage.setItem('token', response.data.token);
    console.log('Logged in:', response.data.user);
  } catch (error) {
    console.error('Login failed:', error.response.data);
  }
}
```

### Task Creation and Management

```javascript
// Admin creates and assigns task
async function createAndAssignTask() {
  try {
    const taskData = {
      title: 'Implement user dashboard',
      description: 'Create responsive dashboard with analytics',
      assignedTo: '507f1f77bcf86cd799439011', // User ID
      priority: 'high',
      dueDate: '2026-05-01'
    };
    
    const response = await taskService.createTask(taskData);
    console.log('Task created:', response.data.task);
  } catch (error) {
    console.error('Task creation failed:', error.response.data);
  }
}

// User updates task status
async function updateTaskProgress(taskId) {
  try {
    await taskService.updateStatus(taskId, 'in-progress');
    console.log('Task status updated');
  } catch (error) {
    console.error('Update failed:', error.response.data);
  }
}
```

### AI-Powered Ticket Creation

```javascript
// User creates support ticket with AI classification
async function createSupportTicket() {
  try {
    const ticketData = {
      title: 'Cannot access admin dashboard',
      description: 'Getting 403 error when trying to access admin features after login',
      priority: 'medium'
    };
    
    // Backend automatically classifies using ML service
    const response = await ticketService.createTicket(ticketData);
    console.log('Ticket created:', response.data.ticket);
    console.log('AI Category:', response.data.ticket.category);
  } catch (error) {
    console.error('Ticket creation failed:', error.response.data);
  }
}
```

### Risk Analysis Dashboard

```javascript
// Admin fetches risk analysis for users
import axios from 'axios';

async function analyzeUserRisk(userId) {
  try {
    const userMetrics = {
      user_id: userId,
      login_frequency: 45,
      failed_logins: 2,
      data_access_volume: 850,
      after_hours_activity: 15
    };
    
    const response = await axios.post(
      `${process.env.REACT_APP_ML_API_URL}/predict-risk`,
      userMetrics
    );
    
    console.log('Risk Analysis:', response.data);
    // { risk_score: 35, risk_level: 'medium', risk_factors: [...] }
  } catch (error) {
    console.error('Risk analysis failed:', error);
  }
}
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

**Common fixes**:
- Ensure MongoDB is running: `sudo systemctl start mongod`
- Check connection string format: `mongodb://localhost:27017/dbname`
- Verify network access if using MongoDB Atlas

### CORS Errors

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3
