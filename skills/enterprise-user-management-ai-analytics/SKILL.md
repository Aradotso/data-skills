---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management built with React, Node.js, and FastAPI ML service
triggers:
  - "setup enterprise user management system"
  - "implement AI analytics for user management"
  - "create user management dashboard with AI insights"
  - "integrate machine learning risk detection in user system"
  - "build task management with burnout detection"
  - "deploy user management system with AI features"
  - "configure user management backend and ML service"
  - "implement JWT authentication for enterprise system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user/task management with AI-powered insights. It provides role-based access control, Kanban-style task boards, support ticket management, and ML-driven features like risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Key Components:**
- **Frontend**: React.js SPA with admin and user dashboards
- **Backend**: Node.js/Express REST API with MongoDB
- **ML Service**: FastAPI microservice using scikit-learn and River for online learning
- **Authentication**: JWT-based secure authentication

## Installation

### Prerequisites
```bash
node --version  # v14+ required
python --version  # Python 3.8+ required
mongod --version  # MongoDB 4.4+ required
```

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

Create `.env` file:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:
```bash
npm start
# or for development
npm run dev
```

### ML Service Setup

```bash
cd ml-service
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

Create `ml-service/.env`:
```env
MODEL_PATH=./models
LOG_LEVEL=info
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `frontend/.env`:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Architecture

### Backend API Structure

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Database connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log('MongoDB connected'))
  .catch(err => console.error('MongoDB connection error:', err));

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/tasks', require('./routes/tasks'));
app.use('/api/tickets', require('./routes/tickets'));
app.use('/api/analytics', require('./routes/analytics'));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  department: String,
  position: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  lastLogin: Date,
  riskScore: { type: Number, default: 0 },
  burnoutScore: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now }
});

UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

UserSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'inprogress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  estimatedTime: Number,
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', TaskSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');
    
    if (!user || user.status !== 'active') {
      return res.status(401).json({ message: 'Invalid token' });
    }

    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Authentication failed' });
  }
};

exports.authorizeAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};
```

### Auth Routes

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'Email already registered' });
    }

    const user = new User({ name, email, password, role });
    await user.save();

    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });

    res.status(201).json({
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Registration failed', error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    user.lastLogin = new Date();
    await user.save();

    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });

    res.json({
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Login failed', error: error.message });
  }
});

module.exports = router;
```

### Task Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authenticate } = require('../middleware/auth');

// Get all tasks for user
router.get('/', authenticate, async (req, res) => {
  try {
    const filter = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user._id };
    
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Failed to fetch tasks', error: error.message });
  }
});

// Create task
router.post('/', authenticate, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
    });
    await task.save();
    await task.populate('assignedTo createdBy', 'name email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Failed to create task', error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authenticate, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    task.status = status;
    task.updatedAt = new Date();
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Failed to update task', error: error.message });
  }
});

// Update time spent
router.patch('/:id/time', authenticate, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeSpent }, updatedAt: new Date() },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Failed to update time', error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI Main Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise UMS ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
class UserBehavior(BaseModel):
    user_id: str
    login_frequency: float
    avg_session_duration: float
    failed_logins: int
    tasks_completed: int
    tasks_overdue: int
    support_tickets: int

class TaskWorkload(BaseModel):
    user_id: str
    active_tasks: int
    total_time_spent: float
    avg_task_completion_time: float
    overdue_tasks: int
    high_priority_tasks: int

class TicketData(BaseModel):
    title: str
    description: str
    priority: Optional[str] = "medium"

# Risk Detection
@app.post("/api/ml/risk-prediction")
async def predict_risk(behavior: UserBehavior):
    """Predict user risk score based on behavior patterns"""
    try:
        # Feature engineering
        features = np.array([
            behavior.login_frequency,
            behavior.avg_session_duration,
            behavior.failed_logins,
            behavior.tasks_completed,
            behavior.tasks_overdue,
            behavior.support_tickets
        ]).reshape(1, -1)
        
        # Simple heuristic model (replace with trained ML model)
        risk_score = calculate_risk_score(features[0])
        
        anomaly = risk_score > 0.7
        
        return {
            "risk_score": float(risk_score),
            "is_anomaly": anomaly,
            "risk_level": get_risk_level(risk_score),
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def calculate_risk_score(features):
    """Calculate risk score using weighted features"""
    weights = np.array([0.1, 0.15, 0.3, -0.15, 0.25, 0.15])
    normalized = (features - features.mean()) / (features.std() + 1e-8)
    score = np.dot(normalized, weights)
    return 1 / (1 + np.exp(-score))  # Sigmoid

def get_risk_level(score):
    if score < 0.3:
        return "low"
    elif score < 0.7:
        return "medium"
    return "high"

# Burnout Detection
@app.post("/api/ml/burnout-detection")
async def detect_burnout(workload: TaskWorkload):
    """Detect employee burnout based on workload metrics"""
    try:
        # Calculate burnout indicators
        overdue_ratio = workload.overdue_tasks / max(workload.active_tasks, 1)
        workload_intensity = workload.total_time_spent / max(workload.active_tasks, 1)
        
        burnout_score = (
            0.4 * min(workload.active_tasks / 20, 1.0) +
            0.3 * overdue_ratio +
            0.2 * min(workload.high_priority_tasks / 10, 1.0) +
            0.1 * min(workload_intensity / 480, 1.0)  # 8 hours
        )
        
        return {
            "burnout_score": float(burnout_score),
            "risk_level": "high" if burnout_score > 0.7 else "medium" if burnout_score > 0.4 else "low",
            "recommendations": get_burnout_recommendations(burnout_score),
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_burnout_recommendations(score):
    if score > 0.7:
        return [
            "Redistribute workload immediately",
            "Schedule mandatory breaks",
            "Review task priorities"
        ]
    elif score > 0.4:
        return [
            "Monitor workload closely",
            "Consider task delegation",
            "Encourage work-life balance"
        ]
    return ["Continue current workload management"]

# Ticket Classification
@app.post("/api/ml/ticket-classification")
async def classify_ticket(ticket: TicketData):
    """Classify support ticket and suggest routing"""
    try:
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Simple keyword-based classification
        if any(word in text for word in ['password', 'login', 'access', 'authentication']):
            category = "authentication"
            department = "IT Security"
        elif any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = "technical"
            department = "Engineering"
        elif any(word in text for word in ['account', 'billing', 'payment']):
            category = "billing"
            department = "Finance"
        else:
            category = "general"
            department = "Support"
        
        # Priority adjustment
        urgent_keywords = ['urgent', 'critical', 'emergency', 'asap']
        suggested_priority = "high" if any(word in text for word in urgent_keywords) else ticket.priority
        
        return {
            "category": category,
            "department": department,
            "suggested_priority": suggested_priority,
            "confidence": 0.85,
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Project Delay Prediction
@app.post("/api/ml/project-prediction")
async def predict_project_delay(data: dict):
    """Predict likelihood of project delay"""
    try:
        tasks_total = data.get('tasks_total', 0)
        tasks_completed = data.get('tasks_completed', 0)
        tasks_overdue = data.get('tasks_overdue', 0)
        days_remaining = data.get('days_remaining', 1)
        
        completion_rate = tasks_completed / max(tasks_total, 1)
        overdue_rate = tasks_overdue / max(tasks_total, 1)
        
        delay_probability = (
            0.5 * (1 - completion_rate) +
            0.3 * overdue_rate +
            0.2 * min(tasks_overdue / max(days_remaining, 1), 1.0)
        )
        
        return {
            "delay_probability": float(delay_probability),
            "will_delay": delay_probability > 0.6,
            "completion_rate": float(completion_rate),
            "estimated_completion_date": calculate_estimated_completion(data),
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def calculate_estimated_completion(data):
    # Simplified estimation
    from datetime import timedelta
    tasks_remaining = data['tasks_total'] - data['tasks_completed']
    days_per_task = data.get('avg_completion_days', 2)
    estimated_days = tasks_remaining * days_per_task
    return (datetime.utcnow() + timedelta(days=estimated_days)).isoformat()

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Frontend Integration

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const api = axios.create({
  baseURL: API_URL,
});

const mlApi = axios.create({
  baseURL: ML_API_URL,
});

// Add auth token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (userData) => api.post('/auth/register', userData),
  logout: () => localStorage.removeItem('token'),
};

export const userService = {
  getAll: () => api.get('/users'),
  getById: (id) => api.get(`/users/${id}`),
  create: (userData) => api.post('/users', userData),
  update: (id, userData) => api.put(`/users/${id}`, userData),
  delete: (id) => api.delete(`/users/${id}`),
};

export const taskService = {
  getAll: () => api.get('/tasks'),
  create: (taskData) => api.post('/tasks', taskData),
  updateStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  updateTime: (id, timeSpent) => api.patch(`/tasks/${id}/time`, { timeSpent }),
};

export const mlService = {
  predictRisk: (behaviorData) => mlApi.post('/api/ml/risk-prediction', behaviorData),
  detectBurnout: (workloadData) => mlApi.post('/api/ml/burnout-detection', workloadData),
  classifyTicket: (ticketData) => mlApi.post('/api/ml/ticket-classification', ticketData),
  predictDelay: (projectData) => mlApi.post('/api/ml/project-prediction', projectData),
};

export default api;
```

### React Task Board Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';
import './TaskBoard.css';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskService.getAll();
      const grouped = groupTasksByStatus(response.data);
      setTasks(grouped);
      setLoading(false);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
      setLoading(false);
    }
  };

  const groupTasksByStatus = (taskList) => {
    return {
      todo: taskList.filter(t => t.status === 'todo'),
      inprogress: taskList.filter(t => t.status === 'inprogress'),
      done: taskList.filter(t => t.status === 'done'),
    };
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskService.updateStatus(taskId, newStatus);
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task status:', error);
    }
  };

  const TaskColumn = ({ title, status, tasks }) => (
    <div className="task-column">
      <h3>{title} ({tasks.length})</h3>
      {tasks.map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <div className="task-meta">
            <span className={`priority-${task.priority}`}>{task.priority}</span>
            <span>{task.assignedTo?.name}</span>
          </div>
          <select 
            value={task.status} 
            onChange={(e) => handleStatusChange(task._id, e.target.value)}
          >
            <option value="todo">To Do</option>
            <option value="inprogress">In Progress</option>
            <option value="done">Done</option>
          </select>
        </div>
      ))}
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="task-board">
      <TaskColumn title="To Do" status="todo" tasks={tasks.todo} />
      <TaskColumn title="In Progress" status="inprogress" tasks={tasks.inprogress} />
      <TaskColumn title="Done" status="done" tasks={tasks.done} />
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { mlService } from '../services/api';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch user behavior data
      const behaviorData = {
        user_id: userId,
        login_frequency: 5.2,
        avg_session_duration: 120,
        failed_logins: 1,
        tasks_completed: 15,
        tasks_overdue: 2,
        support_tickets: 1,
      };

      const riskResponse = await mlService.predictRisk(behaviorData);
      
      const workloadData = {
        user_id: userId,
        active_tasks: 8,
        total_time_spent: 2400,
        avg_task_completion_time: 180,
        overdue_tasks: 2,
        high_priority_tasks: 3,
      };

      const burnoutResponse = await mlService.detectBurnout(workloadData);

      setAnalytics({
        risk: riskResponse.data,
        burnout: burnoutResponse.data,
      });
      setLoading(false);
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h3>Risk Assessment</h3>
        <div className={`score risk-${analytics.risk.risk_level}`}>
          {(analytics.risk.risk_score * 100).toFixed(0)}%
        </div>
        <p>Risk Level: {analytics.risk.risk_level}</p>
        {analytics.risk.is_anomaly && (
          <div className="alert">⚠️ Anomaly detected</div>
        )}
      </div>

      <div className="metric-card">
        <h3>Burnout Detection</h3>
        <div className={`score burnout-${analytics.burnout.risk_level}`}>
          {(analytics.burnout.burnout_score * 100).toFixed(0)}%
        </div>
        <p>Risk Level: {analytics.burnout.risk_level}</p>
        <ul>
          {analytics.burnout.recommendations.map((rec, idx) => (
            <li key={idx}>{rec}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Common Patterns

### Protected Route Component

```javascript
// frontend/src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requireAdmin && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Time Tracker Hook

```javascript
// frontend/src/hooks/useTimeTracker.js
import { useState, useEffect, useRef } from 'react';
import { taskService } from '../services/api';

export const useTimeTracker = (taskId) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else {
      clearInterval(intervalRef.current);
    }

    return () => clearInterval(intervalRef.current);
  }, [isRunning]);

  const start = () => setIsRunning(true);
  
  const stop = async () => {
    setIsRunning(false);
    if (seconds > 0 && taskId) {
      const minutes = Math.floor(seconds / 60);
      await taskService.updateTime(taskId, minutes);
    }
  };

  const reset = () => {
    setIsRunning(false);
    setSeconds(0);
  };

  return {
    seconds,
    isRunning,
    start,
    stop,
    reset,
    formattedTime: formatTime(seconds),
  };
};

const formatTime = (totalSeconds) => {
  const hours = Math.floor(totalSeconds / 3600);
  const minutes = Math.floor((totalSeconds % 3600) / 60);
  const seconds = totalSeconds % 60;
  return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
};
```

## Configuration

### Environment Variables

**Backend (.env):**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your_secure_random_string_min_32_chars
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
NODE_ENV=production
```

**ML Service (.env):**
```env
MODEL_PATH=./models
LOG_LEVEL=info
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
MAX_WORKERS=4
```

**Frontend (.env):**
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

## Deployment

### Production Build

```bash
# Backend
cd backend
npm run build  # if using TypeScript
NODE_ENV=production npm start

# Frontend
cd frontend
npm run build
# Serve build folder with nginx or similar

# ML Service
cd ml-service
gunicorn -w 4 -k uvicorn.workers
