---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management for enterprise organizations
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement user management system with task tracking"
  - "create admin dashboard with AI insights"
  - "build user management with risk detection"
  - "integrate AI analytics for user management"
  - "deploy enterprise user management system"
  - "configure AI-powered ticket classification"
  - "implement kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack web application for managing users, tasks, and support tickets in organizations. It combines React frontend, Node.js backend, MongoDB database, and FastAPI-based ML service to provide intelligent insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

**Key capabilities:**
- User authentication with JWT
- Role-based access control (Admin/User)
- Kanban task boards with time tracking
- Support ticket management
- AI-powered ticket classification and routing
- Risk prediction and anomaly detection
- Burnout analysis and project delay prediction

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB running locally or remotely
- Git

### Clone and Setup

```bash
# Clone the repository
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
MONGODB_URI=mongodb://localhost:27017/enterprise_users
JWT_SECRET=your_jwt_secret_key_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file for ML service
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

ML service runs at `http://localhost:8000`

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

Frontend runs at `http://localhost:3000`

## Architecture

### Backend API Structure

```javascript
// backend/server.js - Main entry point
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const jwt = require('jsonwebtoken');
require('dotenv').config();

const app = express();
app.use(cors());
app.use(express.json());

// MongoDB connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log('MongoDB connected'))
  .catch(err => console.error('MongoDB connection error:', err));

// Routes
const authRoutes = require('./routes/auth');
const userRoutes = require('./routes/users');
const taskRoutes = require('./routes/tasks');
const ticketRoutes = require('./routes/tickets');
const analyticsRoutes = require('./routes/analytics');

app.use('/api/auth', authRoutes);
app.use('/api/users', userRoutes);
app.use('/api/tasks', taskRoutes);
app.use('/api/tickets', ticketRoutes);
app.use('/api/analytics', analyticsRoutes);

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

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
    enum: ['admin', 'user'],
    default: 'user'
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  lastLogin: Date,
  tasksCompleted: {
    type: Number,
    default: 0
  },
  riskScore: {
    type: Number,
    default: 0,
    min: 0,
    max: 100
  },
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
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['todo', 'inprogress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  dueDate: Date,
  timeTracked: {
    type: Number,
    default: 0 // in minutes
  },
  tags: [String],
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
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
    enum: ['technical', 'access', 'general', 'bug', 'feature'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  raisedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  aiClassification: {
    category: String,
    confidence: Number,
    suggestedAssignee: String
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Authentication & Middleware

### JWT Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId).select('-password');
    
    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }

    req.user = user;
    req.userId = decoded.userId;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid authentication token' });
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

### Auth Routes

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const User = require('../models/User');
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
      expiresIn: '7d'
    });

    res.status(201).json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
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
      expiresIn: '7d'
    });

    res.json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { authMiddleware, adminOnly } = require('../middleware/auth');
const router = express.Router();

// Get user tasks
router.get('/my-tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.userId })
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get all tasks (admin)
router.get('/all', authMiddleware, adminOnly, async (req, res) => {
  try {
    const tasks = await Task.find()
      .populate('assignedTo', 'username email')
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      assignedBy: req.userId
    });
    await task.save();
    await task.populate('assignedTo', 'username email');
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

    if (task.assignedTo.toString() !== req.userId && req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Not authorized' });
    }

    task.status = status;
    if (status === 'done' && !task.completedAt) {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time on task
router.patch('/:id/track-time', authMiddleware, async (req, res) => {
  try {
    const { minutes } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    if (task.assignedTo.toString() !== req.userId) {
      return res.status(403).json({ error: 'Not authorized' });
    }

    task.timeTracked += minutes;
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service (FastAPI)

### Main ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from datetime import datetime
import joblib
from pathlib import Path

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models directory
MODEL_DIR = Path("models")
MODEL_DIR.mkdir(exist_ok=True)

# Request models
class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    
class RiskPredictionRequest(BaseModel):
    userId: str
    tasksCompleted: int
    tasksOverdue: int
    avgTaskTime: float
    loginFrequency: int
    lastLogin: str

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    tasksCount: int
    avgWorkHours: float
    overtimeHours: float
    tasksOverdue: int

class AnomalyDetectionRequest(BaseModel):
    userId: str
    loginTime: str
    location: Optional[str] = None
    device: Optional[str] = None

# Ticket classification
@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """AI-based ticket classification and routing"""
    text = f"{request.title} {request.description}".lower()
    
    # Simple keyword-based classification (can be replaced with ML model)
    categories = {
        'technical': ['error', 'bug', 'crash', 'not working', 'broken'],
        'access': ['access', 'permission', 'login', 'password', 'locked'],
        'feature': ['feature', 'request', 'enhancement', 'add', 'new'],
        'bug': ['bug', 'issue', 'problem', 'wrong', 'incorrect']
    }
    
    scores = {}
    for category, keywords in categories.items():
        score = sum(1 for keyword in keywords if keyword in text)
        if score > 0:
            scores[category] = score
    
    if not scores:
        category = 'general'
        confidence = 0.5
    else:
        category = max(scores, key=scores.get)
        total = sum(scores.values())
        confidence = scores[category] / total
    
    # Route based on category
    routing = {
        'technical': 'tech-team',
        'access': 'it-support',
        'feature': 'product-team',
        'bug': 'dev-team',
        'general': 'support-team'
    }
    
    return {
        'category': category,
        'confidence': round(confidence, 2),
        'suggestedTeam': routing.get(category, 'support-team'),
        'priority': 'high' if 'urgent' in text or 'critical' in text else 'medium'
    }

# Risk prediction
@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior patterns"""
    # Calculate risk factors
    overdue_ratio = request.tasksOverdue / max(request.tasksCompleted, 1)
    time_efficiency = request.avgTaskTime / 120.0  # Normalized to 2 hours
    
    # Days since last login
    last_login = datetime.fromisoformat(request.lastLogin.replace('Z', '+00:00'))
    days_since_login = (datetime.now() - last_login).days
    
    # Risk calculation
    risk_score = 0
    risk_score += min(overdue_ratio * 30, 30)  # Max 30 points for overdue
    risk_score += min(time_efficiency * 20, 20)  # Max 20 points for slow tasks
    risk_score += min(days_since_login * 5, 25)  # Max 25 points for inactivity
    risk_score += max(0, (7 - request.loginFrequency) * 3.5)  # Max 25 points for low frequency
    
    risk_score = min(risk_score, 100)
    
    # Risk level
    if risk_score < 30:
        level = 'low'
    elif risk_score < 60:
        level = 'medium'
    else:
        level = 'high'
    
    # Recommendations
    recommendations = []
    if overdue_ratio > 0.3:
        recommendations.append("High overdue task ratio - consider task redistribution")
    if days_since_login > 7:
        recommendations.append("Low engagement - follow up required")
    if time_efficiency > 1.5:
        recommendations.append("Tasks taking longer than expected - training may help")
    
    return {
        'riskScore': round(risk_score, 2),
        'riskLevel': level,
        'factors': {
            'overdueRatio': round(overdue_ratio, 2),
            'taskEfficiency': round(time_efficiency, 2),
            'daysSinceLogin': days_since_login,
            'loginFrequency': request.loginFrequency
        },
        'recommendations': recommendations
    }

# Burnout detection
@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutAnalysisRequest):
    """Detect potential employee burnout"""
    burnout_score = 0
    
    # Workload factor
    if request.tasksCount > 20:
        burnout_score += min((request.tasksCount - 20) * 2, 30)
    
    # Overtime factor
    if request.avgWorkHours > 8:
        burnout_score += min((request.avgWorkHours - 8) * 5, 30)
    
    burnout_score += min(request.overtimeHours * 2, 20)
    
    # Overdue tasks factor
    if request.tasksOverdue > 5:
        burnout_score += min(request.tasksOverdue * 2, 20)
    
    burnout_score = min(burnout_score, 100)
    
    # Burnout level
    if burnout_score < 30:
        level = 'low'
        action = 'Monitor regularly'
    elif burnout_score < 60:
        level = 'moderate'
        action = 'Consider workload adjustment'
    else:
        level = 'high'
        action = 'Immediate intervention required'
    
    return {
        'burnoutScore': round(burnout_score, 2),
        'burnoutLevel': level,
        'recommendedAction': action,
        'factors': {
            'taskLoad': request.tasksCount,
            'avgWorkHours': request.avgWorkHours,
            'overtimeHours': request.overtimeHours,
            'overdueCount': request.tasksOverdue
        }
    }

# Anomaly detection
@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    anomalies = []
    anomaly_score = 0
    
    # Parse login time
    login_time = datetime.fromisoformat(request.loginTime.replace('Z', '+00:00'))
    hour = login_time.hour
    
    # Unusual login time (outside 6 AM - 10 PM)
    if hour < 6 or hour > 22:
        anomalies.append("Unusual login time detected")
        anomaly_score += 30
    
    # Weekend login detection
    if login_time.weekday() >= 5:
        anomalies.append("Weekend login detected")
        anomaly_score += 20
    
    is_anomalous = anomaly_score > 30
    
    return {
        'isAnomalous': is_anomalous,
        'anomalyScore': anomaly_score,
        'anomalies': anomalies,
        'timestamp': request.loginTime,
        'recommendation': 'Review user activity' if is_anomalous else 'Normal behavior'
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Frontend Components

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import TaskBoard from './TaskBoard';
import './UserDashboard.css';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [loading, setLoading] = useState(true);
  const [stats, setStats] = useState({
    total: 0,
    inProgress: 0,
    completed: 0
  });

  const API_URL = process.env.REACT_APP_API_URL;
  
  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const token = localStorage.getItem('token');
      const headers = { Authorization: `Bearer ${token}` };
      
      const [tasksRes, ticketsRes] = await Promise.all([
        axios.get(`${API_URL}/api/tasks/my-tasks`, { headers }),
        axios.get(`${API_URL}/api/tickets/my-tickets`, { headers })
      ]);

      setTasks(tasksRes.data);
      setTickets(ticketsRes.data);
      
      // Calculate stats
      const total = tasksRes.data.length;
      const inProgress = tasksRes.data.filter(t => t.status === 'inprogress').length;
      const completed = tasksRes.data.filter(t => t.status === 'done').length;
      
      setStats({ total, inProgress, completed });
      setLoading(false);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
      setLoading(false);
    }
  };

  if (loading) return <div className="loading">Loading...</div>;

  return (
    <div className="user-dashboard">
      <h1>My Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Tasks</h3>
          <p className="stat-value">{stats.total}</p>
        </div>
        <div className="stat-card">
          <h3>In Progress</h3>
          <p className="stat-value">{stats.inProgress}</p>
        </div>
        <div className="stat-card">
          <h3>Completed</h3>
          <p className="stat-value">{stats.completed}</p>
        </div>
      </div>

      <TaskBoard tasks={tasks} onUpdate={fetchDashboardData} />
      
      <div className="recent-tickets">
        <h2>My Recent Tickets</h2>
        {tickets.length === 0 ? (
          <p>No tickets raised yet</p>
        ) : (
          <ul>
            {tickets.map(ticket => (
              <li key={ticket._id} className={`ticket-item ${ticket.status}`}>
                <strong>{ticket.title}</strong>
                <span className="ticket-status">{ticket.status}</span>
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Kanban Board Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState } from 'react';
import axios from 'axios';
import './TaskBoard.css';

const TaskBoard = ({ tasks, onUpdate }) => {
  const [activeTimer, setActiveTimer] = useState(null);
  const [timerSeconds, setTimerSeconds] = useState(0);
  
  const API_URL = process.env.REACT_APP_API_URL;
  const token = localStorage.getItem('token');

  const columns = {
    todo: tasks.filter(t => t.status === 'todo'),
    inprogress: tasks.filter(t => t.status === 'inprogress'),
    done: tasks.filter(t => t.status === 'done')
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      onUpdate();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const startTimer = (taskId) => {
    setActiveTimer(taskId);
    setTimerSeconds(0);
    
    const interval = setInterval(() => {
      setTimerSeconds(prev => prev + 1);
    }, 1000);
    
    // Store interval ID
    window.taskTimerInterval = interval;
  };

  const stopTimer = async (taskId) => {
    clearInterval(window.taskTimerInterval);
    
    const minutes = Math.floor(timerSeconds / 60);
    
    try {
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/track-time`,
        { minutes },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      setActiveTimer(null);
      setTimerSeconds(0);
      onUpdate();
    } catch (error) {
      console.error('Error tracking time:', error);
    }
  };

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="task-board">
      {Object.entries(columns).map(([status, columnTasks]) => (
        <div key={status} className="task-column">
          <h3 className="column-title">
            {status.toUpperCase().replace('INPROGRESS', 'IN PROGRESS')}
            <span className="task-count">{columnTasks.length}</span>
          </h3>
          
          <div className="tasks-container">
            {columnTasks.map(task => (
              <div key={task._id} className={`task-card priority-${task.priority}`}>
                <h4>{task.title}</h4>
                <p className="task-description">{task.description}</p>
                
                <div className="task-meta">
                  <span className="priority-badge">{task.priority}</span>
                  {task.dueDate && (
                    <span className="due-date">
                      Due: {new Date(task.dueDate).toLocaleDateString()}
                    </span>
                  )}
                </div>
                
                {task.timeTracked > 0 && (
                  <div className="time-tracked">
                    ⏱ {Math.floor(task.timeTracked / 60)}h {task.timeTracked % 60}m tracked
                  </div>
                )}
                
                {activeTimer === task._id && (
                  <div className="active-timer">
                    {formatTime(timerSeconds)}
                  </div>
                )}
                
                <div className="task-actions">
                  {status !== 'done' && (
                    <>
                      {activeTimer === task._id ? (
                        <button onClick={() => stopTimer(task._id)} className="btn-stop">
                          Stop Timer
                        </button>
                      ) : (
                        <button onClick={() => startTimer(task._id)} className="btn-start">
                          Start Timer
                        </button>
                      )}
                    </>
                  )}
                  
                  {status === 'todo' && (
                    <button onClick={() => updateTaskStatus(task._id, 'inprogress')}>
                      Start
                    </button>
                  )}
                  {status === 'inprogress' && (
                    <button onClick={() => updateTaskStatus(task._id, 'done')}>
                      Complete
                
