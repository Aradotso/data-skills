---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and task optimization
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "integrate ML service for risk prediction"
  - "build admin panel with user management"
  - "add AI ticket classification system"
  - "configure JWT authentication for user system"
  - "implement kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user management, task tracking, and support ticket handling with AI-powered insights. It provides risk detection, anomaly detection, burnout analysis, and predictive project insights using machine learning models.

**Key Capabilities:**
- JWT-based authentication with role-based access control (Admin/User)
- Kanban-style task management with time tracking
- Support ticket system with AI-based classification
- ML-powered analytics (risk prediction, burnout detection, anomaly detection)
- Real-time notifications and audit logging
- RESTful API backend with MongoDB storage

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

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

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
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

Create `.env` file in ml-service directory:

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

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

Access at `http://localhost:3000`

## Architecture

### Backend API Structure (Node.js + Express)

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();

app.use(cors());
app.use(express.json());

// Connect to MongoDB
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
});

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
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  loginAttempts: { type: Number, default: 0 },
  lastLogin: Date,
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

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    next();
  } catch (err) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: `User role ${req.user.role} is not authorized to access this route` 
      });
    }
    next();
  };
};
```

### Authentication Routes

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const router = express.Router();

// Register
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = await User.create({ name, email, password, role });
    
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });
    
    res.status(201).json({ token, user: { id: user._id, name, email, role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
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
      user.loginAttempts += 1;
      await user.save();
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    user.loginAttempts = 0;
    user.lastLogin = new Date();
    await user.save();
    
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });
    
    res.json({ token, user: { id: user._id, name: user.name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Task Model and Routes

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', TaskSchema);
```

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { protect, authorize } = require('../middleware/auth');
const router = express.Router();

// Get user tasks
router.get('/my-tasks', protect, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user._id })
      .populate('assignedBy', 'name email')
      .sort('-createdAt');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task (Admin only)
router.post('/', protect, authorize('admin'), async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      assignedBy: req.user._id
    });
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user._id },
      { status, updatedAt: new Date() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update time spent
router.patch('/:id/time', protect, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user._id },
      { $inc: { timeSpent }, updatedAt: new Date() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Ticket System

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const TicketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { type: String, enum: ['technical', 'billing', 'general', 'urgent'], default: 'general' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: String,
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', TicketSchema);
```

```javascript
// backend/routes/tickets.js
const express = require('express');
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { protect, authorize } = require('../middleware/auth');
const router = express.Router();

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Call ML service for classification
    let aiCategory = 'general';
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        title,
        description
      });
      aiCategory = mlResponse.data.category;
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }
    
    const ticket = await Ticket.create({
      title,
      description,
      category: aiCategory,
      aiClassification: aiCategory,
      createdBy: req.user._id
    });
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get all tickets (Admin)
router.get('/all', protect, authorize('admin'), async (req, res) => {
  try {
    const tickets = await Ticket.find()
      .populate('createdBy', 'name email')
      .populate('assignedTo', 'name email')
      .sort('-createdAt');
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', protect, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user._id })
      .populate('assignedTo', 'name email')
      .sort('-createdAt');
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, drift
import pickle
import os

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models storage
ticket_classifier = None
anomaly_detector = anomaly.HalfSpaceTrees(seed=42)
drift_detector = drift.ADWIN()

class TicketRequest(BaseModel):
    title: str
    description: str

class UserBehavior(BaseModel):
    user_id: str
    login_time: str
    login_attempts: int
    location: Optional[str] = None
    device: Optional[str] = None

class WorkloadData(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    hours_worked: float
    missed_deadlines: int

class RiskPrediction(BaseModel):
    user_id: str
    task_complexity: float
    past_performance: float
    deadline_days: int

@app.on_event("startup")
async def load_models():
    global ticket_classifier
    model_path = os.getenv('MODEL_PATH', './models')
    
    # Initialize or load ticket classifier
    # In production, load pre-trained model
    ticket_classifier = RandomForestClassifier(n_estimators=100, random_state=42)

@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketRequest):
    """AI-based ticket classification"""
    try:
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Simple rule-based classification (replace with trained model)
        if any(word in text for word in ['urgent', 'critical', 'emergency', 'asap']):
            category = 'urgent'
            priority = 'high'
        elif any(word in text for word in ['payment', 'invoice', 'billing', 'charge']):
            category = 'billing'
            priority = 'medium'
        elif any(word in text for word in ['bug', 'error', 'crash', 'not working', 'technical']):
            category = 'technical'
            priority = 'high'
        else:
            category = 'general'
            priority = 'low'
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(behavior: UserBehavior):
    """Detect anomalous user behavior"""
    try:
        # Feature vector: login_attempts, hour_of_day
        from datetime import datetime
        login_hour = datetime.fromisoformat(behavior.login_time).hour
        
        features = {
            'login_attempts': behavior.login_attempts,
            'login_hour': login_hour
        }
        
        # Score the behavior
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomalous = score > 0.7
        risk_level = "high" if score > 0.8 else "medium" if score > 0.5 else "low"
        
        return {
            "is_anomalous": is_anomalous,
            "anomaly_score": float(score),
            "risk_level": risk_level,
            "message": "Suspicious login behavior detected" if is_anomalous else "Normal behavior"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-burnout")
async def predict_burnout(workload: WorkloadData):
    """Predict employee burnout risk"""
    try:
        # Calculate burnout indicators
        completion_rate = workload.tasks_completed / max(workload.tasks_assigned, 1)
        hours_per_task = workload.hours_worked / max(workload.tasks_completed, 1)
        deadline_miss_rate = workload.missed_deadlines / max(workload.tasks_assigned, 1)
        
        # Burnout score calculation
        burnout_score = 0
        
        if workload.hours_worked > 50:
            burnout_score += 0.3
        if completion_rate < 0.6:
            burnout_score += 0.2
        if deadline_miss_rate > 0.3:
            burnout_score += 0.25
        if hours_per_task > 8:
            burnout_score += 0.25
        
        risk_level = "high" if burnout_score > 0.7 else "medium" if burnout_score > 0.4 else "low"
        
        recommendations = []
        if workload.hours_worked > 50:
            recommendations.append("Reduce working hours")
        if deadline_miss_rate > 0.3:
            recommendations.append("Reassign some tasks")
        if completion_rate < 0.6:
            recommendations.append("Provide additional support or training")
        
        return {
            "user_id": workload.user_id,
            "burnout_score": float(burnout_score),
            "risk_level": risk_level,
            "recommendations": recommendations,
            "metrics": {
                "completion_rate": float(completion_rate),
                "hours_per_task": float(hours_per_task),
                "deadline_miss_rate": float(deadline_miss_rate)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(data: RiskPrediction):
    """Predict task completion risk"""
    try:
        # Risk calculation based on multiple factors
        risk_score = 0
        
        # Task complexity factor (0-1)
        if data.task_complexity > 0.7:
            risk_score += 0.4
        elif data.task_complexity > 0.5:
            risk_score += 0.2
        
        # Past performance factor
        if data.past_performance < 0.6:
            risk_score += 0.3
        elif data.past_performance < 0.8:
            risk_score += 0.15
        
        # Deadline pressure
        if data.deadline_days < 2:
            risk_score += 0.3
        elif data.deadline_days < 5:
            risk_score += 0.15
        
        risk_level = "high" if risk_score > 0.6 else "medium" if risk_score > 0.3 else "low"
        delay_probability = min(risk_score * 100, 95)
        
        return {
            "user_id": data.user_id,
            "risk_score": float(risk_score),
            "risk_level": risk_level,
            "delay_probability": float(delay_probability),
            "recommendation": "Assign additional resources" if risk_score > 0.6 else "Monitor progress"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

### ML Service Requirements

```txt
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
scikit-learn==1.3.2
river==0.19.0
numpy==1.26.2
python-dotenv==1.0.0
```

## Frontend Implementation

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_BASE_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_BASE_URL + '/api'
});

// Add auth token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authAPI = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (userData) => api.post('/auth/register', userData)
};

export const userAPI = {
  getAll: () => api.get('/users'),
  getProfile: () => api.get('/users/profile'),
  update: (id, data) => api.put(`/users/${id}`, data),
  delete: (id) => api.delete(`/users/${id}`)
};

export const taskAPI = {
  getMyTasks: () => api.get('/tasks/my-tasks'),
  create: (taskData) => api.post('/tasks', taskData),
  updateStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  updateTime: (id, timeSpent) => api.patch(`/tasks/${id}/time`, { timeSpent })
};

export const ticketAPI = {
  getAll: () => api.get('/tickets/all'),
  getMyTickets: () => api.get('/tickets/my-tickets'),
  create: (ticketData) => api.post('/tickets', ticketData),
  update: (id, data) => api.patch(`/tickets/${id}`, data)
};

export const mlAPI = {
  classifyTicket: (data) => axios.post(`${ML_BASE_URL}/classify-ticket`, data),
  detectAnomaly: (data) => axios.post(`${ML_BASE_URL}/detect-anomaly`, data),
  predictBurnout: (data) => axios.post(`${ML_BASE_URL}/predict-burnout`, data),
  predictRisk: (data) => axios.post(`${ML_BASE_URL}/predict-risk`, data)
};

export default api;
```

### Task Board Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });
  const [activeTimer, setActiveTimer] = useState(null);
  const [timeElapsed, setTimeElapsed] = useState(0);

  useEffect(() => {
    loadTasks();
  }, []);

  useEffect(() => {
    let interval;
    if (activeTimer) {
      interval = setInterval(() => {
        setTimeElapsed(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [activeTimer]);

  const loadTasks = async () => {
    try {
      const response = await taskAPI.getMyTasks();
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status].push(task);
        return acc;
      }, { todo: [], 'in-progress': [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to load tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await taskAPI.updateStatus(taskId, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const startTimer = (taskId) => {
    setActiveTimer(taskId);
    setTimeElapsed(0);
  };

  const stopTimer = async (taskId) => {
    if (activeTimer === taskId) {
      const minutes = Math.floor(timeElapsed / 60);
      await taskAPI.updateTime(taskId, minutes);
      setActiveTimer(null);
      setTimeElapsed(0);
      loadTasks();
    }
  };

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const renderTask = (task) => (
    <div key={task._id} className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <span>Time: {task.timeSpent}m</span>
      </div>
      {task.status !== 'done' && (
        <div className="task-actions">
          {activeTimer === task._id ? (
            <>
              <span>{formatTime(timeElapsed)}</span>
              <button onClick={() => stopTimer(task._id)}>Stop</button>
            </>
          ) : (
            <button onClick={() => startTimer(task._id)}>Start Timer</button>
          )}
          {task.status === 'todo' && (
            <button onClick={() => updateTaskStatus(task._id, 'in-progress')}>
              Start Work
            </button>
          )}
          {task.status === 'in-progress' && (
            <button onClick={() => updateTaskStatus(task._id, 'done')}>
              Complete
            </button>
          )}
        </div>
      )}
    </div>
  );

  return (
    <div className="task-board">
      <div className="column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(renderTask)}
      </div>
      <div className="column">
        <h3>In Progress ({tasks['in-progress'].length})</h3>
        {tasks['in-progress'].map(renderTask)}
      </div>
      <div className="column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(renderTask)}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { mlAPI } from '../services/api';

const AIAnalytics = ({ userId }) => {
  const [burnoutData, setBurnoutData] = useState(null);
  const [riskData, setRiskData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadAnalytics();
  }, [userId]);

  const loadAnalytics = async () => {
    try {
      // Example workload data (in production, fetch from backend)
      const workloadData = {
        user_id: userId,
        tasks_assigned: 15,
        tasks_completed: 10,
        hours_worked: 52,
        missed_deadlines: 3
      };

      const burnoutResponse = await mlAPI.predictBurnout(workloadData);
      setBurnoutData(burnoutResponse.data);

      const riskResponse = await mlAPI.predictRisk({
        user_id: userId,
        task_complexity: 0.8,
        past_performance: 0.7,
        deadline_days: 3
      });
      setRiskData(riskResponse.data);

      setLoading(false);
    } catch (error) {
      console.error('Failed to load analytics:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Analytics</h2>
      
      <div className="analytics-card">
        <h3>Burnout Risk Analysis</h3>
        {burnoutData && (
          <>
            <div className={`risk-indicator ${burnoutData.risk_level}`}>
              Risk Level: {burnoutData.risk_level.toUpperCase()}
            </div>
            <div className="score">
              Burnout Score: {(burnoutData.burnout_score * 100).toFixed(0)}%
            </div>
            <div className="metrics">
              <p>Completion Rate: {(burnoutData.metrics.completion_rate * 100).toFixed(0)}%</p>
