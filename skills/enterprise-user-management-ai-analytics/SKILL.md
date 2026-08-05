---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics dashboard"
  - "create user management system with task tracking"
  - "build admin dashboard with AI insights"
  - "integrate AI ticket classification system"
  - "develop user management with burnout detection"
  - "set up kanban task board with AI analytics"
  - "implement JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user administration, task management, and AI-powered analytics. It provides role-based access control (Admin/User), Kanban-style task boards, support ticket management, and ML-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

**Stack**: React.js frontend, Node.js backend, MongoDB database, FastAPI ML service, JWT authentication

## Installation

### Prerequisites

```bash
# Required: Node.js, Python 3.8+, MongoDB
node --version  # v14+
python --version  # 3.8+
mongod --version
```

### Full System Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env  # Configure environment variables
npm start  # Runs on http://localhost:5000

# ML Service setup (separate terminal)
cd ml-service
pip install -r requirements.txt
cp .env.example .env
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup (separate terminal)
cd frontend
npm install
cp .env.example .env
npm start  # Runs on http://localhost:3000
```

### Environment Configuration

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

**ML Service (.env)**
```bash
PORT=8000
MODEL_PATH=./models
LOG_LEVEL=INFO
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

### Database Models

**User Model**
```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  department: String,
  designation: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

**Task Model**
```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'inProgress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 }, // in seconds
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

**Ticket Model**
```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { type: String, enum: ['technical', 'hr', 'finance', 'general'], default: 'general' },
  priority: { type: String, enum: ['low', 'medium', 'high', 'critical'], default: 'medium' },
  status: { type: String, enum: ['open', 'in_progress', 'resolved', 'closed'], default: 'open' },
  raisedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: {
    category: String,
    priority: String,
    confidence: Number
  },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Authentication System

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user || user.status !== 'active') {
      throw new Error();
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid authentication token' });
  }
};

const adminAuth = async (req, res, next) => {
  try {
    await auth(req, res, () => {
      if (req.user.role !== 'admin') {
        return res.status(403).json({ error: 'Admin access required' });
      }
      next();
    });
  } catch (error) {
    res.status(403).json({ error: 'Admin access required' });
  }
};

module.exports = { auth, adminAuth };
```

### Auth Routes

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const { auth } = require('../middleware/auth');

const router = express.Router();

// Register
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ $or: [{ email }, { username }] });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }

    const user = new User({ username, email, password, role });
    await user.save();

    const token = jwt.sign({ userId: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE || '7d'
    });

    res.status(201).json({
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      },
      token
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
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

    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    user.lastLogin = new Date();
    await user.save();

    const token = jwt.sign({ userId: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE || '7d'
    });

    res.json({
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      },
      token
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get current user
router.get('/me', auth, async (req, res) => {
  res.json({ user: req.user });
});

module.exports = router;
```

## Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { auth } = require('../middleware/auth');

const router = express.Router();

// Get all tasks (user sees only assigned tasks, admin sees all)
router.get('/', auth, async (req, res) => {
  try {
    const filter = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user._id };
    
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json({ tasks });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', auth, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
    });
    await task.save();
    
    const populatedTask = await Task.findById(task._id)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username email');
    
    res.status(201).json({ task: populatedTask });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    // Check permission
    if (req.user.role !== 'admin' && task.assignedTo.toString() !== req.user._id.toString()) {
      return res.status(403).json({ error: 'Not authorized' });
    }
    
    task.status = status;
    task.updatedAt = new Date();
    await task.save();
    
    const updatedTask = await Task.findById(task._id)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username email');
    
    res.json({ task: updatedTask });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time on task
router.patch('/:id/time', auth, async (req, res) => {
  try {
    const { seconds } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    task.timeTracked += seconds;
    task.updatedAt = new Date();
    await task.save();
    
    res.json({ task });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## AI Analytics Integration

### ML Service API

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, compose, preprocessing
import pickle
import os

app = FastAPI(title="Enterprise AI Analytics Service")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models storage
models_dir = os.getenv("MODEL_PATH", "./models")
os.makedirs(models_dir, exist_ok=True)

# Ticket classification model
class TicketData(BaseModel):
    title: str
    description: str
    
class TicketClassification(BaseModel):
    category: str
    priority: str
    confidence: float

@app.post("/api/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketData):
    """Classify support ticket using NLP and ML"""
    try:
        # Simple keyword-based classification (replace with trained model)
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Category classification
        if any(word in text for word in ['password', 'login', 'access', 'software', 'bug']):
            category = 'technical'
            confidence = 0.85
        elif any(word in text for word in ['leave', 'salary', 'policy', 'hr']):
            category = 'hr'
            confidence = 0.80
        elif any(word in text for word in ['expense', 'reimbursement', 'invoice', 'payment']):
            category = 'finance'
            confidence = 0.78
        else:
            category = 'general'
            confidence = 0.60
        
        # Priority classification
        if any(word in text for word in ['urgent', 'critical', 'emergency', 'down', 'broken']):
            priority = 'high'
        elif any(word in text for word in ['important', 'soon', 'asap']):
            priority = 'medium'
        else:
            priority = 'low'
        
        return TicketClassification(
            category=category,
            priority=priority,
            confidence=confidence
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Risk detection
class UserBehavior(BaseModel):
    login_times: List[int]  # Hour of day
    failed_logins: int
    unusual_access: bool
    tasks_completed: int
    tasks_overdue: int

class RiskScore(BaseModel):
    risk_level: str  # low, medium, high
    score: float
    factors: List[str]

@app.post("/api/analyze-risk", response_model=RiskScore)
async def analyze_risk(behavior: UserBehavior):
    """Detect user risk based on behavior patterns"""
    try:
        score = 0.0
        factors = []
        
        # Failed login attempts
        if behavior.failed_logins > 5:
            score += 0.3
            factors.append("Multiple failed login attempts")
        
        # Unusual access patterns
        if behavior.unusual_access:
            score += 0.2
            factors.append("Unusual access patterns detected")
        
        # Task performance
        overdue_ratio = behavior.tasks_overdue / max(behavior.tasks_completed, 1)
        if overdue_ratio > 0.5:
            score += 0.25
            factors.append("High overdue task ratio")
        
        # Login time anomalies (e.g., late night access)
        unusual_hours = sum(1 for hour in behavior.login_times if hour < 6 or hour > 22)
        if unusual_hours > len(behavior.login_times) * 0.3:
            score += 0.25
            factors.append("Unusual login times")
        
        # Determine risk level
        if score >= 0.7:
            risk_level = "high"
        elif score >= 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return RiskScore(
            risk_level=risk_level,
            score=min(score, 1.0),
            factors=factors if factors else ["No risk factors detected"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout detection
class WorkloadData(BaseModel):
    hours_worked: List[float]  # Daily hours for last 7 days
    tasks_assigned: int
    tasks_completed: int
    missed_deadlines: int
    weekend_work_hours: float

class BurnoutAnalysis(BaseModel):
    burnout_risk: str  # low, medium, high
    score: float
    recommendations: List[str]

@app.post("/api/detect-burnout", response_model=BurnoutAnalysis)
async def detect_burnout(workload: WorkloadData):
    """Detect employee burnout risk"""
    try:
        score = 0.0
        recommendations = []
        
        # Average work hours
        avg_hours = np.mean(workload.hours_worked)
        if avg_hours > 10:
            score += 0.4
            recommendations.append("Reduce daily work hours to 8-9 hours")
        elif avg_hours > 8:
            score += 0.2
        
        # Weekend work
        if workload.weekend_work_hours > 5:
            score += 0.3
            recommendations.append("Minimize weekend work")
        
        # Task completion ratio
        completion_ratio = workload.tasks_completed / max(workload.tasks_assigned, 1)
        if completion_ratio < 0.5:
            score += 0.2
            recommendations.append("Reassign some tasks to balance workload")
        
        # Missed deadlines
        if workload.missed_deadlines > 3:
            score += 0.1
            recommendations.append("Review task priorities and deadlines")
        
        # Risk level
        if score >= 0.7:
            burnout_risk = "high"
            recommendations.append("Schedule immediate 1-on-1 with manager")
        elif score >= 0.4:
            burnout_risk = "medium"
            recommendations.append("Monitor workload closely")
        else:
            burnout_risk = "low"
        
        if not recommendations:
            recommendations = ["Workload appears balanced"]
        
        return BurnoutAnalysis(
            burnout_risk=burnout_risk,
            score=min(score, 1.0),
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Project delay prediction
class ProjectData(BaseModel):
    total_tasks: int
    completed_tasks: int
    days_elapsed: int
    total_days: int
    team_size: int
    critical_tasks_pending: int

class DelayPrediction(BaseModel):
    will_delay: bool
    delay_days: int
    confidence: float
    suggestions: List[str]

@app.post("/api/predict-delay", response_model=DelayPrediction)
async def predict_delay(project: ProjectData):
    """Predict project delay"""
    try:
        # Calculate progress ratio
        expected_progress = project.days_elapsed / project.total_days
        actual_progress = project.completed_tasks / project.total_tasks
        progress_gap = expected_progress - actual_progress
        
        # Estimate remaining days needed
        remaining_tasks = project.total_tasks - project.completed_tasks
        if project.completed_tasks > 0:
            avg_tasks_per_day = project.completed_tasks / project.days_elapsed
            estimated_days = remaining_tasks / avg_tasks_per_day
            remaining_days = project.total_days - project.days_elapsed
            delay_days = max(0, int(estimated_days - remaining_days))
        else:
            delay_days = 0
        
        will_delay = progress_gap > 0.15 or delay_days > 0
        confidence = min(0.6 + abs(progress_gap), 0.95)
        
        suggestions = []
        if will_delay:
            if project.team_size < 5 and remaining_tasks > 20:
                suggestions.append("Consider adding more team members")
            if project.critical_tasks_pending > 5:
                suggestions.append("Prioritize critical tasks immediately")
            suggestions.append("Daily standup to identify blockers")
        else:
            suggestions.append("Project on track")
        
        return DelayPrediction(
            will_delay=will_delay,
            delay_days=delay_days,
            confidence=confidence,
            suggestions=suggestions
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "AI Analytics"}
```

### Backend Integration with ML Service

```javascript
// backend/routes/analytics.js
const express = require('express');
const axios = require('axios');
const { auth, adminAuth } = require('../middleware/auth');
const Task = require('../models/Task');
const User = require('../models/User');

const router = express.Router();
const ML_SERVICE_URL = process.env.ML_SERVICE_URL || 'http://localhost:8000';

// Analyze user risk
router.post('/user-risk/:userId', adminAuth, async (req, res) => {
  try {
    const user = await User.findById(req.params.userId);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    // Gather user behavior data
    const tasks = await Task.find({ assignedTo: user._id });
    const overdueTasks = tasks.filter(t => t.dueDate && t.dueDate < new Date() && t.status !== 'done');
    
    const behaviorData = {
      login_times: [9, 10, 14, 15, 16], // Mock data - implement actual tracking
      failed_logins: 0, // Implement login tracking
      unusual_access: false,
      tasks_completed: tasks.filter(t => t.status === 'done').length,
      tasks_overdue: overdueTasks.length
    };

    const mlResponse = await axios.post(`${ML_SERVICE_URL}/api/analyze-risk`, behaviorData);
    
    res.json({
      user: {
        id: user._id,
        username: user.username,
        email: user.email
      },
      risk_analysis: mlResponse.data
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Detect burnout
router.post('/burnout/:userId', adminAuth, async (req, res) => {
  try {
    const user = await User.findById(req.params.userId);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    const tasks = await Task.find({ assignedTo: user._id });
    
    const workloadData = {
      hours_worked: [8, 9, 10, 8.5, 9, 7, 4], // Mock - implement time tracking
      tasks_assigned: tasks.length,
      tasks_completed: tasks.filter(t => t.status === 'done').length,
      missed_deadlines: tasks.filter(t => t.dueDate && t.dueDate < new Date() && t.status !== 'done').length,
      weekend_work_hours: 3 // Mock
    };

    const mlResponse = await axios.post(`${ML_SERVICE_URL}/api/detect-burnout`, workloadData);
    
    res.json({
      user: {
        id: user._id,
        username: user.username
      },
      burnout_analysis: mlResponse.data
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## Frontend Components

### React Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${API_URL}/auth/me`);
      setUser(response.data.user);
    } catch (error) {
      console.error('Failed to fetch user:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/auth/login`, { email, password });
    const { token, user } = response.data;
    
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    
    return user;
  };

  const register = async (userData) => {
    const response = await axios.post(`${API_URL}/auth/register`, userData);
    const { token, user } = response.data;
    
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
    delete axios.defaults.headers.common['Authorization'];
  };

  return (
    <AuthContext.Provider value={{ user, token, login, register, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Task Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/tasks`);
      const allTasks = response.data.tasks;
      
      setTasks({
        todo: allTasks.filter(t => t.status === 'todo'),
        inProgress: allTasks.filter(t => t.status === 'inProgress'),
        done: allTasks.filter(t => t.status === 'done')
      });
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/tasks/${taskId}/status`, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = (e, status) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, status);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const TaskCard = ({ task }) => (
    <div
      className="task-card"
      draggable
      onDragStart={(e) => handleDragStart(e, task._id)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        {task.assignedTo && <span className="assigned">{task.assignedTo.username}</span>}
      </div>
      {task.dueDate && (
        <div className="due-date">Due: {new Date(task.dueDate).toLocaleDateString()}</div>
      )}
    </div>
  );

  if (loading) return <div>Loading...</div>;

  return (
    <div className="kanban-board">
      <div
        className="
