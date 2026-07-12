---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management with AI analytics"
  - "how do I implement AI-based ticket classification"
  - "configure JWT authentication for user management system"
  - "integrate ML service for burnout detection"
  - "create a task management dashboard with kanban board"
  - "implement role-based access control with AI insights"
  - "build user analytics with risk prediction"
  - "deploy user management system with FastAPI ML backend"
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System with AI Analytics - a full-stack application combining user/task management with ML-powered insights including risk detection, anomaly detection, burnout analysis, and intelligent ticket routing.

## What This Project Does

The Enterprise User Management System provides:
- **User & Task Management**: CRUD operations, role-based access, task assignment with Kanban boards
- **AI-Powered Analytics**: Risk prediction, anomaly detection, burnout analysis, ticket classification
- **Support Ticket System**: Smart routing and classification using ML
- **Real-time Tracking**: Time tracking, progress monitoring, notifications
- **Admin Dashboard**: Organization analytics, audit logs, security alerts

**Tech Stack**: React.js frontend, Node.js/Express backend, FastAPI ML service, MongoDB database, JWT authentication

## Installation

### Prerequisites
- Node.js 14+
- Python 3.8+
- MongoDB running locally or accessible instance

### Clone and Setup
```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup
```bash
cd backend
npm install
```

Create `backend/.env`:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
```

Start backend:
```bash
npm start
# Runs on http://localhost:5000
```

### ML Service Setup
```bash
cd ml-service
pip install -r requirements.txt
```

Create `ml-service/.env`:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:
```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Runs on http://localhost:8000
```

### Frontend Setup
```bash
cd frontend
npm install
```

Create `frontend/.env`:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
# Runs on http://localhost:3000
```

## Key API Endpoints

### Authentication (Backend)
```javascript
// POST /api/auth/register - Register new user
// POST /api/auth/login - Login user
// GET /api/auth/me - Get current user (requires JWT)
```

### User Management (Backend)
```javascript
// GET /api/users - Get all users (admin only)
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user (admin only)
// POST /api/users - Create user (admin only)
```

### Task Management (Backend)
```javascript
// GET /api/tasks - Get all tasks
// POST /api/tasks - Create task
// PUT /api/tasks/:id - Update task
// DELETE /api/tasks/:id - Delete task
// PATCH /api/tasks/:id/status - Update task status
```

### Support Tickets (Backend)
```javascript
// GET /api/tickets - Get all tickets
// POST /api/tickets - Create ticket
// PUT /api/tickets/:id - Update ticket
// GET /api/tickets/:id/classify - AI classify ticket
```

### ML Analytics (ML Service)
```python
# POST /api/ml/classify-ticket - Classify support ticket
# POST /api/ml/detect-risk - Predict user risk score
# POST /api/ml/detect-anomaly - Detect anomalous behavior
# POST /api/ml/predict-burnout - Analyze burnout risk
# POST /api/ml/project-insights - Predict project delays
```

## Backend Implementation Patterns

### JWT Authentication Middleware
```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ success: false, message: 'Not authorized' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ success: false, message: 'Token invalid' });
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        success: false, 
        message: 'User role not authorized' 
      });
    }
    next();
  };
};

module.exports = { protect, authorize };
```

### User Controller with Authentication
```javascript
// backend/controllers/authController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

const generateToken = (id) => {
  return jwt.sign({ id }, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_EXPIRE
  });
};

exports.register = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ 
        success: false, 
        message: 'User already exists' 
      });
    }
    
    const user = await User.create({
      name,
      email,
      password,
      role: role || 'user'
    });
    
    const token = generateToken(user._id);
    
    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email }).select('+password');
    
    if (!user || !(await user.matchPassword(password))) {
      return res.status(401).json({ 
        success: false, 
        message: 'Invalid credentials' 
      });
    }
    
    const token = generateToken(user._id);
    
    res.json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

### Task Management with Status Tracking
```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.getTasks = async (req, res) => {
  try {
    const { status, assignedTo } = req.query;
    let filter = {};
    
    if (req.user.role !== 'admin') {
      filter.assignedTo = req.user._id;
    } else if (assignedTo) {
      filter.assignedTo = assignedTo;
    }
    
    if (status) {
      filter.status = status;
    }
    
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort('-createdAt');
    
    res.json({ success: true, count: tasks.length, data: tasks });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user._id
    });
    
    await task.populate('assignedTo', 'name email');
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const validStatuses = ['todo', 'in_progress', 'done'];
    
    if (!validStatuses.includes(status)) {
      return res.status(400).json({ 
        success: false, 
        message: 'Invalid status' 
      });
    }
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true, runValidators: true }
    ).populate('assignedTo', 'name email');
    
    if (!task) {
      return res.status(404).json({ 
        success: false, 
        message: 'Task not found' 
      });
    }
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};
```

### Support Ticket with AI Classification
```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

exports.createTicket = async (req, res) => {
  try {
    const ticket = await Ticket.create({
      ...req.body,
      createdBy: req.user._id,
      status: 'open'
    });
    
    // Auto-classify with ML service
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/ml/classify-ticket`,
        {
          title: ticket.title,
          description: ticket.description
        }
      );
      
      ticket.category = mlResponse.data.category;
      ticket.priority = mlResponse.data.priority;
      await ticket.save();
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    await ticket.populate('createdBy', 'name email');
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};
```

## ML Service Implementation

### FastAPI ML Service Structure
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import joblib
import numpy as np
from datetime import datetime

app = FastAPI(title="Enterprise User Management ML Service")

# Load or initialize models
try:
    ticket_classifier = joblib.load('./models/ticket_classifier.pkl')
except:
    ticket_classifier = None

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: int
    failed_login_attempts: int
    task_completion_rate: float
    avg_response_time: float

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_task_duration: float
    overtime_hours: float
    days_since_last_break: int

@app.get("/")
async def root():
    return {"message": "Enterprise User Management ML Service", "status": "running"}

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket into category and priority"""
    try:
        # Simple rule-based classification (replace with ML model)
        text = f"{request.title} {request.description}".lower()
        
        # Category classification
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            priority = 'high'
        elif any(word in text for word in ['password', 'login', 'access']):
            category = 'access'
            priority = 'medium'
        elif any(word in text for word in ['feature', 'request', 'enhancement']):
            category = 'feature_request'
            priority = 'low'
        else:
            category = 'general'
            priority = 'medium'
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-risk")
async def detect_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior patterns"""
    try:
        # Calculate risk score (0-100)
        risk_score = 0
        
        # Failed login attempts contribute to risk
        if request.failed_login_attempts > 3:
            risk_score += min(request.failed_login_attempts * 10, 40)
        
        # Low task completion rate
        if request.task_completion_rate < 0.5:
            risk_score += 20
        
        # Abnormal login frequency
        if request.login_frequency < 5 or request.login_frequency > 50:
            risk_score += 15
        
        # High response time
        if request.avg_response_time > 300:  # > 5 minutes
            risk_score += 15
        
        risk_score = min(risk_score, 100)
        
        risk_level = 'low' if risk_score < 30 else 'medium' if risk_score < 70 else 'high'
        
        return {
            "user_id": request.user_id,
            "risk_score": risk_score,
            "risk_level": risk_level,
            "factors": {
                "failed_logins": request.failed_login_attempts,
                "task_completion": request.task_completion_rate,
                "login_frequency": request.login_frequency
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-burnout")
async def predict_burnout(request: BurnoutAnalysisRequest):
    """Analyze burnout risk for a user"""
    try:
        burnout_score = 0
        
        # High workload
        workload_ratio = request.tasks_assigned / max(request.tasks_completed, 1)
        if workload_ratio > 1.5:
            burnout_score += 30
        
        # Long task duration
        if request.avg_task_duration > 8:  # > 8 hours
            burnout_score += 25
        
        # Overtime hours
        if request.overtime_hours > 10:
            burnout_score += min(request.overtime_hours * 2, 30)
        
        # No breaks
        if request.days_since_last_break > 30:
            burnout_score += 15
        
        burnout_score = min(burnout_score, 100)
        
        risk_level = 'low' if burnout_score < 40 else 'medium' if burnout_score < 70 else 'high'
        
        recommendations = []
        if burnout_score >= 40:
            recommendations.append("Reduce task assignments")
        if request.days_since_last_break > 30:
            recommendations.append("Schedule time off")
        if request.overtime_hours > 10:
            recommendations.append("Limit overtime hours")
        
        return {
            "user_id": request.user_id,
            "burnout_score": burnout_score,
            "risk_level": risk_level,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Integration Patterns

### API Service Configuration
```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add auth token to requests
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

export const authAPI = {
  login: (credentials) => api.post('/api/auth/login', credentials),
  register: (userData) => api.post('/api/auth/register', userData),
  getMe: () => api.get('/api/auth/me')
};

export const taskAPI = {
  getTasks: (params) => api.get('/api/tasks', { params }),
  createTask: (taskData) => api.post('/api/tasks', taskData),
  updateTask: (id, taskData) => api.put(`/api/tasks/${id}`, taskData),
  updateStatus: (id, status) => api.patch(`/api/tasks/${id}/status`, { status }),
  deleteTask: (id) => api.delete(`/api/tasks/${id}`)
};

export const mlAPI = {
  classifyTicket: (ticketData) => 
    axios.post(`${ML_API_URL}/api/ml/classify-ticket`, ticketData),
  detectRisk: (userData) => 
    axios.post(`${ML_API_URL}/api/ml/detect-risk`, userData),
  predictBurnout: (userData) => 
    axios.post(`${ML_API_URL}/api/ml/predict-burnout`, userData)
};

export default api;
```

### React Task Dashboard Component
```javascript
// frontend/src/components/TaskDashboard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';

const TaskDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getTasks();
      const tasksByStatus = {
        todo: response.data.data.filter(t => t.status === 'todo'),
        in_progress: response.data.data.filter(t => t.status === 'in_progress'),
        done: response.data.data.filter(t => t.status === 'done')
      };
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskAPI.updateStatus(taskId, newStatus);
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task status:', error);
    }
  };

  const renderTaskColumn = (status, title) => (
    <div className="task-column">
      <h3>{title}</h3>
      <div className="task-list">
        {tasks[status].map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <select 
              value={task.status} 
              onChange={(e) => handleStatusChange(task._id, e.target.value)}
            >
              <option value="todo">To Do</option>
              <option value="in_progress">In Progress</option>
              <option value="done">Done</option>
            </select>
          </div>
        ))}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {renderTaskColumn('todo', 'To Do')}
      {renderTaskColumn('in_progress', 'In Progress')}
      {renderTaskColumn('done', 'Done')}
    </div>
  );
};

export default TaskDashboard;
```

## Database Models

### User Model (MongoDB/Mongoose)
```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please add a name'],
    trim: true
  },
  email: {
    type: String,
    required: [true, 'Please add an email'],
    unique: true,
    match: [/^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/, 'Please add a valid email']
  },
  password: {
    type: String,
    required: [true, 'Please add a password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: {
    type: String,
    default: ''
  },
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: {
    type: Date
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    next();
  }
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

// Match password method
UserSchema.methods.matchPassword = async function(enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Task Model
```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, 'Please add a task title'],
    trim: true,
    maxlength: [100, 'Title cannot be more than 100 characters']
  },
  description: {
    type: String,
    required: [true, 'Please add a description'],
    maxlength: [500, 'Description cannot be more than 500 characters']
  },
  status: {
    type: String,
    enum: ['todo', 'in_progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  assignedTo: {
    type: mongoose.Schema.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: {
    type: Date
  },
  timeTracked: {
    type: Number,
    default: 0  // in minutes
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', TaskSchema);
```

## Configuration

### Environment Variables

**Backend (.env)**:
```env
PORT=5000
NODE_ENV=development
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_secure_jwt_secret_min_32_chars
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
```

**ML Service (.env)**:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5000
```

**Frontend (.env)**:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Common Patterns

### Protected Route with Role Check
```javascript
// backend/routes/userRoutes.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const { getUsers, createUser, updateUser, deleteUser } = require('../controllers/userController');

router.route('/')
  .get(protect, authorize('admin'), getUsers)
  .post(protect, authorize('admin'), createUser);

router.route('/:id')
  .put(protect, updateUser)
  .delete(protect, authorize('admin'), deleteUser);

module.exports = router;
```

### AI-Powered Analytics Dashboard
```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { mlAPI } from '../services/api';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    highRiskUsers: [],
    burnoutAlerts: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch user data and analyze with ML
      const users = await getUserData(); // Your API call
      
      const riskAnalysis = await Promise.all(
        users.map(user => mlAPI.detectRisk({
          user_id: user.id,
          login_frequency: user.loginCount,
          failed_login_attempts: user.failedLogins,
          task_completion_rate: user.completionRate,
          avg_response_time: user.avgResponseTime
        }))
      );
      
      const highRisk = riskAnalysis
        .filter(r => r.data.risk_level === 'high')
        .map(r => r.data);
      
      setAnalytics(prev => ({ ...prev, highRiskUsers: highRisk }));
    } catch (error) {
      console.error('Analytics fetch failed:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h2>AI-Powered Analytics</h2>
      <div className="alert-section">
        <h3>High Risk Users ({analytics.highRiskUsers.length})</h3>
        {analytics.highRiskUsers.map(user => (
          <div key={user.user_id} className="alert-card">
            <p>User: {user.user_id}</p>
            <p>Risk Score: {user.risk_score}</p>
            <p>Factors: {JSON.stringify(user.factors)}</p>
          </div>
        ))}
      </div>
    </div>
  );
};
```

## Troubleshooting

### JWT Token Issues
```javascript
// Check token validity
const token = localStorage.getItem('token');
if (!token) {
  // Redirect to login
  window.location.href = '/login';
}

// Verify token format
const parts = token.split('.');
if (parts.length !== 3) {
  console.error('Invalid JWT format');
  localStorage.removeItem('token');
}
```

### MongoDB Connection Error
```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Service Not Responding
```python
# Check ML service health
import requests

try:
    response = requests.get('http://localhost:8000/')
    print(f"ML Service Status: {response.json()}")
except requests.exceptions.ConnectionError:
    print("ML Service is not running. Start with: uvicorn main:app --reload")
```

### CORS Issues
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

```python
# ml-service/main.py
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"]
)
```

### Task Status Update Not Working
```javascript
// Ensure proper task ID format
const updateTaskStatus = async (taskId, newStatus) => {
  // Validate MongoDB ObjectId
  if (!taskId.match(/^[0-9a-fA-F]{24}$/)) {
    console.error('Invalid task ID format');
    return;
  }
  
  try {
    const response = await taskAPI.updateStatus(taskId, newStatus);
    console.log('Task updated:', response.data);
  } catch (error) {
    console.error('Update failed:', error.response?.data || error.message);
  }
};
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI agents to assist with implementation, integration, troubleshooting, and extension of this full-stack application.
