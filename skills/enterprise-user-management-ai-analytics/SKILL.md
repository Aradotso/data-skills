---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "configure AI analytics for user management"
  - "implement risk detection and anomaly detection"
  - "create user management dashboard with AI insights"
  - "integrate machine learning service with user management"
  - "build task tracking with burnout detection"
  - "add AI-powered ticket classification"
  - "deploy user management system with FastAPI ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack JavaScript application that combines traditional user/task management with AI-powered analytics. It provides risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing using machine learning models integrated via FastAPI.

**Key Components:**
- **Frontend**: React.js dashboard with Kanban boards and real-time analytics
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI with scikit-learn and River for online learning
- **Database**: MongoDB for user, task, and ticket storage

## Installation

### Prerequisites
```bash
# Node.js 16+ and Python 3.8+
node --version
python --version
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
MONGODB_URI=mongodb://localhost:27017/user_management
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
```

Start backend:
```bash
npm start
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
uvicorn main:app --reload --port 8000
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
```

## Architecture

### Backend API Structure (Node.js)

**Authentication Middleware:**
```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authenticateToken = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'Access denied' });
  }
  
  try {
    const verified = jwt.verify(token, process.env.JWT_SECRET);
    req.user = verified;
    next();
  } catch (error) {
    res.status(400).json({ error: 'Invalid token' });
  }
};

module.exports = authenticateToken;
```

**User Model (MongoDB):**
```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  department: String,
  tasksCompleted: { type: Number, default: 0 },
  totalWorkHours: { type: Number, default: 0 },
  riskScore: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  lastActive: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);
```

**Task Model:**
```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
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
  estimatedHours: Number,
  actualHours: Number,
  dueDate: Date,
  createdAt: { type: Date, default: Date.now },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

**Ticket Model:**
```javascript
// models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { 
    type: String, 
    enum: ['technical', 'access', 'general', 'urgent'],
    default: 'general'
  },
  priority: { type: String, enum: ['low', 'medium', 'high'] },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassified: { type: Boolean, default: false },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### Backend API Endpoints

**Authentication:**
```javascript
// routes/auth.js
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const router = express.Router();

// Register
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    res.status(201).json({ message: 'User created successfully' });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const validPassword = await bcrypt.compare(password, user.password);
    if (!validPassword) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({ 
      token, 
      user: { 
        id: user._id, 
        username: user.username, 
        role: user.role 
      } 
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

**Task Management:**
```javascript
// routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const authenticateToken = require('../middleware/auth');
const axios = require('axios');

const router = express.Router();

// Get all tasks (admin) or user tasks
router.get('/', authenticateToken, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user.id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authenticateToken, async (req, res) => {
  try {
    const task = new Task(req.body);
    await task.save();
    
    // Trigger AI prediction for project delay
    if (task.estimatedHours) {
      try {
        const prediction = await axios.post(
          `${process.env.ML_SERVICE_URL}/predict/project-delay`,
          {
            estimatedHours: task.estimatedHours,
            taskCount: 1,
            userId: task.assignedTo
          }
        );
        task.delayRisk = prediction.data.risk_score;
        await task.save();
      } catch (mlError) {
        console.error('ML prediction failed:', mlError.message);
      }
    }
    
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authenticateToken, async (req, res) => {
  try {
    const { status, actualHours } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    task.status = status;
    if (actualHours) task.actualHours = actualHours;
    if (status === 'done') task.completedAt = new Date();
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

**Ticket Management with AI:**
```javascript
// routes/tickets.js
const express = require('express');
const Ticket = require('../models/Ticket');
const authenticateToken = require('../middleware/auth');
const axios = require('axios');

const router = express.Router();

// Create ticket with AI classification
router.post('/', authenticateToken, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Call ML service for ticket classification
    let category = 'general';
    let priority = 'medium';
    
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify/ticket`,
        { title, description }
      );
      
      category = mlResponse.data.category;
      priority = mlResponse.data.priority;
    } catch (mlError) {
      console.error('AI classification failed:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      category,
      priority,
      createdBy: req.user.id,
      aiClassified: true
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get tickets
router.get('/', authenticateToken, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.id };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

**Analytics and Risk Detection:**
```javascript
// routes/analytics.js
const express = require('express');
const User = require('../models/User');
const Task = require('../models/Task');
const authenticateToken = require('../middleware/auth');
const axios = require('axios');

const router = express.Router();

// Admin only middleware
const isAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

// Get risk analysis for user
router.get('/risk/:userId', authenticateToken, isAdmin, async (req, res) => {
  try {
    const user = await User.findById(req.params.userId);
    const tasks = await Task.find({ assignedTo: req.params.userId });
    
    const inProgressTasks = tasks.filter(t => t.status === 'in-progress').length;
    const overdueTasks = tasks.filter(t => 
      t.dueDate && new Date(t.dueDate) < new Date() && t.status !== 'done'
    ).length;
    
    // Call ML service for risk prediction
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/predict/risk`,
      {
        userId: req.params.userId,
        totalWorkHours: user.totalWorkHours,
        tasksCompleted: user.tasksCompleted,
        inProgressTasks,
        overdueTasks
      }
    );
    
    res.json({
      user: {
        id: user._id,
        username: user.username,
        email: user.email
      },
      riskScore: mlResponse.data.risk_score,
      factors: mlResponse.data.factors,
      recommendation: mlResponse.data.recommendation
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Burnout detection
router.get('/burnout/:userId', authenticateToken, isAdmin, async (req, res) => {
  try {
    const user = await User.findById(req.params.userId);
    const tasks = await Task.find({ 
      assignedTo: req.params.userId,
      createdAt: { $gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) }
    });
    
    const avgHoursPerTask = tasks.length > 0
      ? tasks.reduce((sum, t) => sum + (t.actualHours || 0), 0) / tasks.length
      : 0;
    
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/predict/burnout`,
      {
        userId: req.params.userId,
        weeklyHours: user.totalWorkHours / 52, // Approximate
        tasksPerWeek: tasks.length / 4,
        avgTaskHours: avgHoursPerTask
      }
    );
    
    res.json(mlResponse.data);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Anomaly detection
router.post('/anomaly', authenticateToken, isAdmin, async (req, res) => {
  try {
    const { userId, action, timestamp } = req.body;
    
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/detect/anomaly`,
      { userId, action, timestamp }
    );
    
    res.json(mlResponse.data);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### ML Service (FastAPI)

**Main Application:**
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, compose, preprocessing
import joblib
import os

app = FastAPI(title="User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
risk_model = None
burnout_model = None
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

# Load or initialize models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")
os.makedirs(MODEL_PATH, exist_ok=True)

try:
    risk_model = joblib.load(f"{MODEL_PATH}/risk_model.pkl")
except:
    risk_model = RandomForestClassifier(n_estimators=100)

try:
    burnout_model = joblib.load(f"{MODEL_PATH}/burnout_model.pkl")
except:
    burnout_model = RandomForestClassifier(n_estimators=100)

# Request models
class RiskPredictionRequest(BaseModel):
    userId: str
    totalWorkHours: float
    tasksCompleted: int
    inProgressTasks: int
    overdueTasks: int

class BurnoutPredictionRequest(BaseModel):
    userId: str
    weeklyHours: float
    tasksPerWeek: float
    avgTaskHours: float

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class AnomalyDetectionRequest(BaseModel):
    userId: str
    action: str
    timestamp: str

# Risk prediction
@app.post("/predict/risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        features = np.array([[
            request.totalWorkHours,
            request.tasksCompleted,
            request.inProgressTasks,
            request.overdueTasks
        ]])
        
        # Simple rule-based scoring (can be replaced with trained model)
        risk_score = 0
        factors = []
        
        if request.totalWorkHours > 200:
            risk_score += 30
            factors.append("High total work hours")
        
        if request.overdueTasks > 3:
            risk_score += 40
            factors.append("Multiple overdue tasks")
        
        if request.inProgressTasks > 5:
            risk_score += 20
            factors.append("High number of concurrent tasks")
        
        completion_rate = request.tasksCompleted / max(request.totalWorkHours / 10, 1)
        if completion_rate < 0.5:
            risk_score += 10
            factors.append("Low task completion rate")
        
        risk_score = min(risk_score, 100)
        
        recommendation = "Low risk" if risk_score < 30 else \
                        "Medium risk - monitor workload" if risk_score < 60 else \
                        "High risk - intervention needed"
        
        return {
            "risk_score": risk_score,
            "factors": factors,
            "recommendation": recommendation
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout prediction
@app.post("/predict/burnout")
async def predict_burnout(request: BurnoutPredictionRequest):
    try:
        burnout_score = 0
        indicators = []
        
        if request.weeklyHours > 50:
            burnout_score += 35
            indicators.append("Working excessive hours")
        
        if request.tasksPerWeek > 15:
            burnout_score += 30
            indicators.append("High task volume")
        
        if request.avgTaskHours > 8:
            burnout_score += 25
            indicators.append("Complex/lengthy tasks")
        
        # Check work-life balance
        if request.weeklyHours > 45 and request.tasksPerWeek > 10:
            burnout_score += 10
            indicators.append("Poor work-life balance")
        
        burnout_score = min(burnout_score, 100)
        
        status = "Low risk" if burnout_score < 30 else \
                "Moderate risk - schedule breaks" if burnout_score < 60 else \
                "High risk - immediate action needed"
        
        return {
            "burnout_score": burnout_score,
            "status": status,
            "indicators": indicators,
            "recommendations": [
                "Reduce weekly hours" if request.weeklyHours > 50 else None,
                "Delegate tasks" if request.tasksPerWeek > 15 else None,
                "Break down complex tasks" if request.avgTaskHours > 8 else None
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Ticket classification
@app.post("/classify/ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Rule-based classification
        category = "general"
        priority = "medium"
        
        # Category detection
        if any(word in text for word in ["bug", "error", "crash", "broken", "not working"]):
            category = "technical"
            priority = "high"
        elif any(word in text for word in ["access", "permission", "login", "password"]):
            category = "access"
            priority = "high"
        elif any(word in text for word in ["urgent", "critical", "asap", "emergency"]):
            category = "urgent"
            priority = "high"
        
        # Priority adjustment
        if "urgent" in text or "critical" in text:
            priority = "high"
        elif "low priority" in text or "whenever" in text:
            priority = "low"
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Anomaly detection
@app.post("/detect/anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        # Create feature vector from action
        features = {
            'hour': int(request.timestamp.split('T')[1].split(':')[0]) if 'T' in request.timestamp else 12,
            'action_type': hash(request.action) % 100
        }
        
        # Update anomaly detector
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.7
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": float(score),
            "userId": request.userId,
            "action": request.action,
            "alert": "Suspicious activity detected" if is_anomaly else "Normal activity"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Project delay prediction
@app.post("/predict/project-delay")
async def predict_project_delay(request: dict):
    try:
        estimated = request.get("estimatedHours", 0)
        task_count = request.get("taskCount", 1)
        
        # Simple heuristic
        risk_score = 0
        
        if estimated > 40:
            risk_score += 30
        
        if task_count > 10:
            risk_score += 20
        
        # Add complexity factor
        complexity = estimated / task_count if task_count > 0 else 0
        if complexity > 8:
            risk_score += 25
        
        return {
            "risk_score": min(risk_score, 100),
            "prediction": "Likely delay" if risk_score > 50 else "On track"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

### Frontend Integration (React)

**API Service:**
```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_URL,
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authAPI = {
  login: (credentials) => api.post('/api/auth/login', credentials),
  register: (userData) => api.post('/api/auth/register', userData),
};

export const taskAPI = {
  getTasks: () => api.get('/api/tasks'),
  createTask: (taskData) => api.post('/api/tasks', taskData),
  updateTaskStatus: (id, status, actualHours) => 
    api.patch(`/api/tasks/${id}/status`, { status, actualHours }),
  deleteTask: (id) => api.delete(`/api/tasks/${id}`),
};

export const ticketAPI = {
  getTickets: () => api.get('/api/tickets'),
  createTicket: (ticketData) => api.post('/api/tickets', ticketData),
  updateTicket: (id, updates) => api.patch(`/api/tickets/${id}`, updates),
};

export const analyticsAPI = {
  getRiskAnalysis: (userId) => api.get(`/api/analytics/risk/${userId}`),
  getBurnoutAnalysis: (userId) => api.get(`/api/analytics/burnout/${userId}`),
  detectAnomaly: (data) => api.post('/api/analytics/anomaly', data),
};

export default api;
```

**Task Dashboard Component:**
```javascript
// frontend/src/components/TaskDashboard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI, analyticsAPI } from '../services/api';

const TaskDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadTasks();
    loadBurnoutAnalysis();
  }, []);

  const loadTasks = async () => {
    try {
      const response = await taskAPI.getTasks();
      setTasks(response.data);
    } catch (error) {
      console.error('Failed to load tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const loadBurnoutAnalysis = async () => {
    try {
      const userId = JSON.parse(localStorage.getItem('user')).id;
      const response = await analyticsAPI.getBurnoutAnalysis(userId);
      setBurnoutData(response.data);
    } catch (error) {
      console.error('Failed to load burnout analysis:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await taskAPI.updateTaskStatus(taskId, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const getStatusColor = (status) => {
    switch (status) {
      case 'todo': return '#gray';
      case 'in-progress': return '#blue';
      case 'done': return '#green';
      default: return '#gray';
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="task-dashboard">
      <h2>My Tasks</h2>
      
      {burnoutData && burnoutData.burnout_score > 60 && (
        <div className="alert alert-warning">
          ⚠️ Burnout Risk: {burnoutData.status}
          <p>{burnoutData.indicators.join(', ')}</p>
        </div>
      )}
      
      <div className="kanban-board">
        {['todo', 'in-progress', 'done'].map(status => (
          <div key={status} className="kanban-column">
            <h3>{status.toUpperCase()}</h3>
            {tasks
              .filter(task => task.status === status)
              .map(task => (
                <div 
                  key={task._id} 
                  className="task-card"
                  style={{ borderLeft: `4px solid ${getStatusColor(status)}` }}
                >
                  <h4>{task.title}</h4>
                  <p>{task.description}</p>
                  <div className="task-meta">
                    <span>Priority: {task.priority}</span>
                    {task.estimatedHours && (
                      <span>Est: {task.estimatedHours}h</span>
                    )}
                  </div>
                  <select 
                    value={task.status}
                    onChange={(e) => updateTaskStatus(task._id, e.target.value)}
                  >
                    <option value="todo">To Do</option>
                    <option value="in-progress">In Progress</option>
                    <option value="done">Done</option>
                  </select>
                </div>
              ))}
          </div>
        ))}
      </div>
    </div>
  );
};

export default TaskDashboard;
```

**Admin Analytics Dashboard:**
```javascript
// frontend/src/components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { analyticsAPI } from '../services/api';

const AdminAnalytics = () => {
  const [users, setUsers] = useState([]);
  const [selectedUser, setSelectedUser] = useState(null);
  const [riskData, setRiskData] = useState(null);

  const analyzeUser = async (userId) => {
    try {
      const response = await analyticsAPI.getRiskAnalysis(userId);
      setRiskData(response.data);
      setSelectedUser(userId);
    } catch (error) {
      console.error('Risk analysis failed:', error);
    }
  };

  const getRiskColor = (score) => {
    if (score < 30) return '#28a745';
    if (score < 60) return '#ffc107';
    return '#dc3545';
  };

  return (
    <div className="admin-analytics">
      <h2>User Risk Analytics</h2>
      
      <div className="user-list">
        {users.map(user => (
          <button 
            key={user.id}
            onClick={() => analyze
