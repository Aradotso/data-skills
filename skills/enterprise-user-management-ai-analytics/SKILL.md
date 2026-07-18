---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, and risk detection
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "create a user management dashboard with AI insights"
  - "implement task tracking with burnout detection"
  - "set up JWT authentication for user management"
  - "build an AI-powered ticket classification system"
  - "configure the ML service for anomaly detection"
  - "deploy enterprise user management with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack application that combines user administration, task management, and support ticketing with AI-powered analytics. It provides risk detection, anomaly detection, burnout analysis, and predictive insights using machine learning models deployed via FastAPI.

**Key Capabilities:**
- User authentication and role-based access control (JWT)
- Task management with Kanban board and time tracking
- Support ticket system with AI classification
- ML-powered risk prediction and anomaly detection
- Burnout detection based on workload patterns
- Admin dashboard with organization analytics

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB running locally or remote connection string

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup (Node.js)

```bash
cd backend
npm install
```

Create `.env` file:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:
```bash
npm start
```

### ML Service Setup (Python/FastAPI)

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:
```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:
```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install
```

Create `.env` file:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Architecture

**Three-tier architecture:**
1. **Frontend (React)** - User interface on port 3000
2. **Backend (Node.js)** - API server on port 5000
3. **ML Service (FastAPI)** - AI analytics on port 8000

## Backend API (Node.js)

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const router = express.Router();

// Register new user
router.post('/register', async (req, res) => {
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
    res.status(201).json({ message: 'User registered successfully' });
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
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, role: user.role } });
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
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

const router = express.Router();

// Get user tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('assignedBy', 'username')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.userId,
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
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Tickets with AI Integration

```javascript
// backend/routes/tickets.js
const express = require('express');
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authMiddleware } = require('../middleware/auth');

const router = express.Router();

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { subject, description, priority } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
      subject,
      description
    });
    
    const ticket = new Ticket({
      subject,
      description,
      priority: priority || mlResponse.data.suggestedPriority,
      category: mlResponse.data.category,
      createdBy: req.user.userId,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.userId })
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service (FastAPI/Python)

### Main Application Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Optional
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models (create placeholder if not exists)
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    suggestedPriority: str
    confidence: float

@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and suggest priority"""
    try:
        # Simple keyword-based classification
        text = f"{request.subject} {request.description}".lower()
        
        # Category detection
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            priority = 'high'
        elif any(word in text for word in ['feature', 'enhancement', 'improve']):
            category = 'feature_request'
            priority = 'medium'
        elif any(word in text for word in ['password', 'login', 'access']):
            category = 'access'
            priority = 'high'
        else:
            category = 'general'
            priority = 'low'
        
        return TicketClassificationResponse(
            category=category,
            suggestedPriority=priority,
            confidence=0.85
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Risk Detection

```python
# ml-service/main.py (continued)
class UserBehaviorRequest(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    taskCompletionRate: float
    avgTaskDuration: float
    ticketsCreated: int

class RiskDetectionResponse(BaseModel):
    riskScore: float
    riskLevel: str
    factors: List[str]

@app.post("/detect-risk", response_model=RiskDetectionResponse)
async def detect_risk(request: UserBehaviorRequest):
    """Detect risk based on user behavior patterns"""
    try:
        risk_score = 0.0
        factors = []
        
        # Failed login attempts
        if request.failedLogins > 5:
            risk_score += 30
            factors.append("Multiple failed login attempts")
        
        # Low task completion rate
        if request.taskCompletionRate < 0.5:
            risk_score += 20
            factors.append("Low task completion rate")
        
        # Excessive tickets
        if request.ticketsCreated > 10:
            risk_score += 15
            factors.append("High number of support tickets")
        
        # Determine risk level
        if risk_score >= 50:
            risk_level = "high"
        elif risk_score >= 25:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return RiskDetectionResponse(
            riskScore=min(risk_score, 100),
            riskLevel=risk_level,
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Burnout Detection

```python
# ml-service/main.py (continued)
class BurnoutAnalysisRequest(BaseModel):
    userId: str
    tasksAssigned: int
    tasksCompleted: int
    avgWorkHoursPerDay: float
    overtimeHours: float
    lastBreakDays: int

class BurnoutAnalysisResponse(BaseModel):
    burnoutScore: float
    burnoutLevel: str
    recommendations: List[str]

@app.post("/analyze-burnout", response_model=BurnoutAnalysisResponse)
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze employee burnout risk"""
    try:
        burnout_score = 0.0
        recommendations = []
        
        # Workload analysis
        if request.tasksAssigned - request.tasksCompleted > 10:
            burnout_score += 25
            recommendations.append("Redistribute tasks to reduce workload")
        
        # Working hours
        if request.avgWorkHoursPerDay > 9:
            burnout_score += 30
            recommendations.append("Reduce daily working hours")
        
        # Overtime
        if request.overtimeHours > 20:
            burnout_score += 20
            recommendations.append("Limit overtime work")
        
        # Breaks
        if request.lastBreakDays > 30:
            burnout_score += 25
            recommendations.append("Schedule a break or vacation")
        
        # Determine level
        if burnout_score >= 60:
            burnout_level = "critical"
        elif burnout_score >= 40:
            burnout_level = "high"
        elif burnout_score >= 20:
            burnout_level = "moderate"
        else:
            burnout_level = "low"
        
        return BurnoutAnalysisResponse(
            burnoutScore=min(burnout_score, 100),
            burnoutLevel=burnout_level,
            recommendations=recommendations if recommendations else ["No immediate action needed"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Anomaly Detection

```python
# ml-service/main.py (continued)
from river import anomaly
from datetime import datetime

# Online learning anomaly detector
anomaly_detector = anomaly.HalfSpaceTrees(seed=42)

class AnomalyDetectionRequest(BaseModel):
    userId: str
    loginTime: str
    loginLocation: str
    deviceType: str
    actionsPerformed: int

class AnomalyDetectionResponse(BaseModel):
    isAnomaly: bool
    anomalyScore: float
    reason: Optional[str]

@app.post("/detect-anomaly", response_model=AnomalyDetectionResponse)
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    try:
        # Convert to feature vector
        hour = int(request.loginTime.split(':')[0])
        features = {
            'hour': hour,
            'actions': request.actionsPerformed,
            'is_mobile': 1 if request.deviceType == 'mobile' else 0
        }
        
        # Score anomaly
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.7
        reason = None
        
        if is_anomaly:
            if hour < 6 or hour > 22:
                reason = "Unusual login time"
            elif request.actionsPerformed > 100:
                reason = "Excessive actions performed"
            else:
                reason = "Unusual behavior pattern detected"
        
        return AnomalyDetectionResponse(
            isAnomaly=is_anomaly,
            anomalyScore=float(score),
            reason=reason
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend (React)

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_URL = process.env.REACT_APP_ML_SERVICE_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_URL
});

const mlApi = axios.create({
  baseURL: ML_URL
});

// Add token to requests
api.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (userData) => api.post('/auth/register', userData)
};

export const taskService = {
  getTasks: () => api.get('/tasks'),
  createTask: (task) => api.post('/tasks', task),
  updateStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status })
};

export const ticketService = {
  createTicket: (ticket) => api.post('/tickets', ticket),
  getMyTickets: () => api.get('/tickets/my-tickets')
};

export const mlService = {
  detectRisk: (userData) => mlApi.post('/detect-risk', userData),
  analyzeBurnout: (userData) => mlApi.post('/analyze-burnout', userData),
  detectAnomaly: (behaviorData) => mlApi.post('/detect-anomaly', behaviorData)
};

export default api;
```

### Task Component with Kanban

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';
import './TaskBoard.css';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    try {
      const response = await taskService.getTasks();
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Error loading tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await taskService.updateStatus(taskId, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-badge ${task.priority}`}>{task.priority}</span>
      <div className="task-actions">
        {task.status !== 'in_progress' && (
          <button onClick={() => moveTask(task._id, 'in_progress')}>
            Start
          </button>
        )}
        {task.status !== 'done' && (
          <button onClick={() => moveTask(task._id, 'done')}>
            Complete
          </button>
        )}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>Done</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
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
import { mlService } from '../services/api';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    risk: null,
    burnout: null
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Example data - replace with real user metrics
      const riskData = {
        userId,
        loginAttempts: 15,
        failedLogins: 2,
        taskCompletionRate: 0.75,
        avgTaskDuration: 120,
        ticketsCreated: 3
      };

      const burnoutData = {
        userId,
        tasksAssigned: 20,
        tasksCompleted: 15,
        avgWorkHoursPerDay: 8.5,
        overtimeHours: 10,
        lastBreakDays: 15
      };

      const [riskResponse, burnoutResponse] = await Promise.all([
        mlService.detectRisk(riskData),
        mlService.analyzeBurnout(burnoutData)
      ]);

      setAnalytics({
        risk: riskResponse.data,
        burnout: burnoutResponse.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="ai-analytics">
      {analytics.risk && (
        <div className={`analytics-card risk-${analytics.risk.riskLevel}`}>
          <h3>Risk Assessment</h3>
          <div className="score">{analytics.risk.riskScore.toFixed(1)}</div>
          <p className="level">Level: {analytics.risk.riskLevel}</p>
          <ul>
            {analytics.risk.factors.map((factor, idx) => (
              <li key={idx}>{factor}</li>
            ))}
          </ul>
        </div>
      )}

      {analytics.burnout && (
        <div className={`analytics-card burnout-${analytics.burnout.burnoutLevel}`}>
          <h3>Burnout Analysis</h3>
          <div className="score">{analytics.burnout.burnoutScore.toFixed(1)}</div>
          <p className="level">Level: {analytics.burnout.burnoutLevel}</p>
          <h4>Recommendations:</h4>
          <ul>
            {analytics.burnout.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Database Models (MongoDB/Mongoose)

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
  },
  email: {
    type: String,
    required: true,
    unique: true
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
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date,
  isActive: {
    type: Boolean,
    default: true
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
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
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
  dueDate: Date,
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  },
  timeSpent: {
    type: Number,
    default: 0
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'feature_request', 'access', 'general'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in_progress', 'resolved', 'closed'],
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
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### Admin Dashboard Analytics

```javascript
// backend/routes/admin.js
const express = require('express');
const User = require('../models/User');
const Task = require('../models/Task');
const Ticket = require('../models/Ticket');
const { authMiddleware, adminOnly } = require('../middleware/auth');

const router = express.Router();

router.get('/analytics', authMiddleware, adminOnly, async (req, res) => {
  try {
    const [
      totalUsers,
      activeUsers,
      totalTasks,
      completedTasks,
      openTickets
    ] = await Promise.all([
      User.countDocuments(),
      User.countDocuments({ isActive: true }),
      Task.countDocuments(),
      Task.countDocuments({ status: 'done' }),
      Ticket.countDocuments({ status: { $in: ['open', 'in_progress'] } })
    ]);

    const taskCompletionRate = totalTasks > 0 
      ? (completedTasks / totalTasks * 100).toFixed(2) 
      : 0;

    res.json({
      users: { total: totalUsers, active: activeUsers },
      tasks: { total: totalTasks, completed: completedTasks, completionRate: taskCompletionRate },
      tickets: { open: openTickets }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Time Tracking

```javascript
// frontend/src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId, onSave }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(sec => sec + 1);
      }, 1000);
    } else if (!isRunning && seconds !== 0) {
      clearInterval(interval);
    }
    return () => clearInterval(interval);
  }, [isRunning, seconds]);

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const handleSave = () => {
    onSave(taskId, seconds);
    setSeconds(0);
    setIsRunning(false);
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <div className="timer-controls">
        <button onClick={() => setIsRunning(!isRunning)}>
          {isRunning ? 'Pause' : 'Start'}
        </button>
        <button onClick={() => { setSeconds(0); setIsRunning(false); }}>
          Reset
        </button>
        <button onClick={handleSave}>Save</button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

## Configuration

### Environment Variables

**Backend (.env):**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_strong_secret_key_here
JWT_EXPIRATION=24h
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env):**
```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
RIVER_SEED=42
```

**Frontend (.env):**
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

##
