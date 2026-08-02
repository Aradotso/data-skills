---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management for enterprise applications
triggers:
  - "implement user management with AI analytics"
  - "create enterprise task tracking system"
  - "add AI-powered risk detection to user management"
  - "build admin dashboard with user analytics"
  - "setup kanban board with time tracking"
  - "integrate AI ticket classification system"
  - "develop user management with JWT authentication"
  - "create AI burnout detection for users"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with AI-powered insights. It provides role-based access control, task tracking with Kanban boards, support ticket management, and AI features including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Key Components:**
- **Frontend**: React.js with JWT authentication
- **Backend**: Node.js with REST APIs
- **ML Service**: FastAPI with scikit-learn and River for online learning
- **Database**: MongoDB
- **AI Features**: Ticket classification, risk prediction, anomaly detection, burnout analysis

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file (if needed)
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Backend API Structure

### User Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }
    
    const user = new User({ username, email, password, role: role || 'user' });
    await user.save();
    
    const token = jwt.sign({ id: user._id, role: user.role }, process.env.JWT_SECRET, {
      expiresIn: '24h'
    });
    
    res.json({ token, user: { id: user._id, username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user || !(await user.comparePassword(password))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign({ id: user._id, role: user.role }, process.env.JWT_SECRET, {
      expiresIn: '24h'
    });
    
    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Middleware for Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ error: 'No token provided' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

// Get user tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task (admin only)
router.post('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.id,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Not authorized' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time on task
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body; // duration in seconds
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Ticket API

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authMiddleware } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    let category = 'general';
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        text: `${title} ${description}`
      });
      category = mlResponse.data.category;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      category,
      priority: priority || 'medium',
      createdBy: req.user.id,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get tickets
router.get('/', authMiddleware, async (req, res) => {
  try {
    const query = req.user.role === 'admin' ? {} : { createdBy: req.user.id };
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## MongoDB Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

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
    enum: ['user', 'admin'],
    default: 'user'
  },
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
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
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  dueDate: Date,
  timeSpent: {
    type: Number,
    default: 0
  },
  completedAt: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise User Management AI Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Request/Response models
class TicketClassificationRequest(BaseModel):
    text: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_logins: int
    unusual_hours_activity: int
    data_access_volume: float
    permission_changes: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_task_duration: float
    overtime_hours: float
    
class AnomalyDetectionRequest(BaseModel):
    user_id: str
    login_time: str
    ip_address: str
    action: str
    resource_accessed: str

# Ticket classification
@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket into categories"""
    text_lower = request.text.lower()
    
    # Simple rule-based classification (can be replaced with ML model)
    if any(word in text_lower for word in ['password', 'login', 'access', 'permission']):
        category = 'authentication'
    elif any(word in text_lower for word in ['bug', 'error', 'crash', 'not working']):
        category = 'technical'
    elif any(word in text_lower for word in ['feature', 'request', 'add', 'improve']):
        category = 'feature_request'
    elif any(word in text_lower for word in ['slow', 'performance', 'timeout']):
        category = 'performance'
    else:
        category = 'general'
    
    return {
        "category": category,
        "confidence": 0.85
    }

# Risk prediction
@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior"""
    # Calculate risk score (0-100)
    risk_score = 0
    
    # Failed login attempts
    if request.failed_logins > 5:
        risk_score += 30
    elif request.failed_logins > 2:
        risk_score += 15
    
    # Unusual hours activity
    if request.unusual_hours_activity > 10:
        risk_score += 25
    elif request.unusual_hours_activity > 5:
        risk_score += 10
    
    # High data access
    if request.data_access_volume > 1000:
        risk_score += 20
    elif request.data_access_volume > 500:
        risk_score += 10
    
    # Frequent permission changes
    if request.permission_changes > 3:
        risk_score += 25
    
    risk_level = 'low'
    if risk_score > 60:
        risk_level = 'high'
    elif risk_score > 30:
        risk_level = 'medium'
    
    return {
        "user_id": request.user_id,
        "risk_score": min(risk_score, 100),
        "risk_level": risk_level,
        "factors": {
            "failed_logins": request.failed_logins,
            "unusual_hours": request.unusual_hours_activity,
            "data_access": request.data_access_volume,
            "permission_changes": request.permission_changes
        }
    }

# Burnout analysis
@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze user burnout risk"""
    burnout_score = 0
    
    # Task completion rate
    completion_rate = request.tasks_completed / max(request.tasks_assigned, 1)
    if completion_rate < 0.5:
        burnout_score += 30
    elif completion_rate < 0.7:
        burnout_score += 15
    
    # Average task duration (if too high, might indicate struggle)
    if request.avg_task_duration > 8:  # hours
        burnout_score += 25
    elif request.avg_task_duration > 5:
        burnout_score += 10
    
    # Overtime hours
    if request.overtime_hours > 20:
        burnout_score += 30
    elif request.overtime_hours > 10:
        burnout_score += 15
    
    # Workload
    if request.tasks_assigned > 20:
        burnout_score += 15
    
    burnout_level = 'low'
    if burnout_score > 60:
        burnout_level = 'high'
    elif burnout_score > 30:
        burnout_level = 'medium'
    
    recommendations = []
    if burnout_level in ['medium', 'high']:
        recommendations.append("Reduce task assignments")
        if request.overtime_hours > 10:
            recommendations.append("Limit overtime hours")
        if completion_rate < 0.7:
            recommendations.append("Provide additional support or training")
    
    return {
        "user_id": request.user_id,
        "burnout_score": min(burnout_score, 100),
        "burnout_level": burnout_level,
        "completion_rate": round(completion_rate, 2),
        "recommendations": recommendations
    }

# Anomaly detection
@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    is_anomaly = False
    anomaly_reasons = []
    
    # Check login time
    try:
        login_hour = datetime.fromisoformat(request.login_time.replace('Z', '+00:00')).hour
        if login_hour < 6 or login_hour > 22:
            is_anomaly = True
            anomaly_reasons.append("Unusual login time")
    except:
        pass
    
    # Check suspicious actions
    suspicious_actions = ['delete_all', 'mass_export', 'privilege_escalation']
    if any(action in request.action.lower() for action in suspicious_actions):
        is_anomaly = True
        anomaly_reasons.append("Suspicious action detected")
    
    # Check resource access patterns
    sensitive_resources = ['admin_panel', 'user_database', 'financial_data']
    if any(resource in request.resource_accessed.lower() for resource in sensitive_resources):
        if login_hour < 6 or login_hour > 22:
            is_anomaly = True
            anomaly_reasons.append("Sensitive resource access at unusual time")
    
    return {
        "user_id": request.user_id,
        "is_anomaly": is_anomaly,
        "anomaly_score": len(anomaly_reasons) * 33,
        "reasons": anomaly_reasons,
        "timestamp": datetime.utcnow().isoformat()
    }

# Project delay prediction
class ProjectPredictionRequest(BaseModel):
    tasks_total: int
    tasks_completed: int
    days_elapsed: int
    days_remaining: int
    team_size: int

@app.post("/predict-project-delay")
async def predict_project_delay(request: ProjectPredictionRequest):
    """Predict if project will be delayed"""
    completion_rate = request.tasks_completed / max(request.tasks_total, 1)
    expected_completion = (request.days_elapsed / max(request.days_elapsed + request.days_remaining, 1))
    
    delay_probability = 0
    
    if completion_rate < expected_completion - 0.1:
        delay_probability = 0.7
    elif completion_rate < expected_completion:
        delay_probability = 0.4
    else:
        delay_probability = 0.2
    
    # Adjust for team size
    if request.team_size < 3 and request.tasks_total > 50:
        delay_probability += 0.1
    
    is_delayed = delay_probability > 0.5
    
    return {
        "is_delayed": is_delayed,
        "delay_probability": min(delay_probability, 1.0),
        "completion_rate": round(completion_rate, 2),
        "expected_completion": round(expected_completion, 2),
        "recommendations": [
            "Increase team size" if request.team_size < 3 else None,
            "Re-prioritize tasks" if completion_rate < 0.5 else None,
            "Review task complexity" if delay_probability > 0.7 else None
        ]
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Frontend React Components

### Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      // Verify token and get user info
      fetchUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    
    localStorage.setItem('token', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
    setUser(response.data.user);
    
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`);
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        'in-progress': response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable onDragStart={(e) => e.dataTransfer.setData('taskId', task._id)}>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        {task.dueDate && <span>Due: {new Date(task.dueDate).toLocaleDateString()}</span>}
      </div>
      {task.timeSpent > 0 && (
        <div className="time-spent">
          Time: {Math.floor(task.timeSpent / 3600)}h {Math.floor((task.timeSpent % 3600) / 60)}m
        </div>
      )}
    </div>
  );

  const handleDrop = (e, status) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, status);
  };

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {Object.entries(tasks).map(([status, statusTasks]) => (
        <div
          key={status}
          className="kanban-column"
          onDragOver={(e) => e.preventDefault()}
          onDrop={(e) => handleDrop(e, status)}
        >
          <h3>{status.replace('-', ' ').toUpperCase()}</h3>
          <div className="task-list">
            {statusTasks.map(task => (
              <TaskCard key={task._id} task={task} />
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Time Tracker Component

```javascript
// frontend/src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const [startTime, setStartTime] = useState(null);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setElapsedTime(Math.floor((Date.now() - startTime) / 1000));
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning, startTime]);

  const startTimer = () => {
    setStartTime(Date.now());
    setIsRunning(true);
  };

  const stopTimer = async () => {
    setIsRunning(false);
    
    // Save time to backend
    try {
      await axios.post(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`,
        { duration: elapsedTime }
      );
      setElapsedTime(0);
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };

  const formatTime = (seconds) => {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    const s = seconds % 60;
    return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(elapsedTime)}</div>
      <div className="timer-controls">
        {!isRunning ? (
          <button onClick={startTimer} className="btn-start">Start</button>
        ) : (
          <button onClick={stopTimer} className="btn-stop">Stop & Save</button>
        )}
      </div>
    </div>
  );
};

export default TimeTracker;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const [analyticsRes, usersRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/admin/analytics`),
        axios.get(`${process.env.REACT_APP_API_URL}/api/admin/users`)
      ]);
      
      setAnalytics(analyticsRes.data);
      
      // Get risk scores for users
      const riskPromises = usersRes.data.map(user =>
        axios.post(`${process.env.REACT_APP_ML_SERVICE_URL}/predict-risk`, {
          user_id: user._id,
          failed_logins: user.failedLogins || 0,
          unusual_hours_activity: user.unusualHoursActivity || 0,
          data_access_volume: user.dataAccessVolume || 0,
          permission_changes: user.permissionChanges || 0
        }).catch(() => ({ data: { risk_level: 'low', risk_score: 0 } }))
      );
      
      const riskResults = await Promise.all(riskPromises);
      const usersWithRisk = usersRes.data.map((user, index) => ({
        ...user,
        ...riskResults[index].data
      })).filter(u => u.risk_level !== 'low');
      
      setRiskUsers(usersWithRisk);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading dashboard...</div>;

  return (
    <div className
