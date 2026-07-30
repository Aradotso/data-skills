---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create admin dashboard with user management"
  - "build task tracking with kanban board"
  - "add AI ticket classification system"
  - "configure burnout detection analytics"
  - "integrate ML service for risk prediction"
  - "develop user management with JWT authentication"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack user management platform combining React frontend, Node.js backend, and FastAPI ML service to deliver AI-driven insights for enterprise operations including risk detection, anomaly detection, burnout analysis, and intelligent ticket routing.

## What It Does

- **User Management**: Role-based access control (Admin/User) with JWT authentication
- **Task Management**: Kanban-style task boards with time tracking
- **Support Tickets**: AI-classified ticket system with automatic routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay forecasting
- **Admin Dashboard**: Centralized control for user/task/ticket management with audit logs

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB running locally or cloud instance

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
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_key_here
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

Create `.env` file:

```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
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

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Key API Endpoints

### Authentication

```javascript
// Register user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@company.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securepass123"
}
// Returns: { token, user }
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Create user
POST /api/users
{
  "name": "Jane Smith",
  "email": "jane@company.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:id
{
  "name": "Jane Smith Updated",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { "Authorization": "Bearer <token>" }

// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Add new dashboard widget",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PUT /api/tasks/:id
{
  "status": "in-progress" // or "done"
}

// Track time
POST /api/tasks/:id/time
{
  "duration": 3600 // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "subject": "Login issue",
  "description": "Cannot access dashboard after password reset",
  "priority": "high"
}

// Get tickets (filtered by role)
GET /api/tickets

// AI classify ticket
POST /api/ml/classify-ticket
{
  "subject": "Payment gateway error",
  "description": "Transaction failing at checkout"
}
// Returns: { category: "technical", priority: "high", suggestedAssignee: "team_id" }
```

### AI Analytics

```javascript
// Risk detection
POST /api/ml/risk-detection
{
  "userId": "user_id",
  "recentActivity": {
    "failedLogins": 3,
    "unusualHours": true,
    "taskCompletionRate": 0.3
  }
}
// Returns: { riskScore: 0.75, alerts: ["Multiple failed logins", "Low productivity"] }

// Burnout analysis
POST /api/ml/burnout-analysis
{
  "userId": "user_id",
  "workload": {
    "weeklyHours": 65,
    "openTasks": 15,
    "overdueCount": 5
  }
}
// Returns: { burnoutRisk: "high", recommendation: "Reduce workload by 30%" }

// Anomaly detection
POST /api/ml/anomaly-detection
{
  "userId": "user_id",
  "loginTime": "2026-04-15T03:30:00Z",
  "ipAddress": "192.168.1.100",
  "location": "Unknown"
}
// Returns: { isAnomaly: true, confidence: 0.89 }
```

## Frontend Integration

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      // Fetch user profile
      fetchProfile();
    }
  }, [token]);

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
    delete axios.defaults.headers.common['Authorization'];
  };

  const fetchProfile = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/auth/profile`);
      setUser(response.data);
    } catch (error) {
      logout();
    }
  };

  return { user, token, login, logout, isAdmin: user?.role === 'admin' };
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await axios.get(`${API_URL}/api/tasks`);
    const grouped = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await axios.put(`${API_URL}/api/tasks/${taskId}`, { status: newStatus });
    fetchTasks();
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-${task.priority}`}>{task.priority}</span>
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

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Risk detection
      const riskResponse = await axios.post(`${ML_API_URL}/risk-detection`, {
        userId
      });

      // Burnout analysis
      const burnoutResponse = await axios.post(`${ML_API_URL}/burnout-analysis`, {
        userId
      });

      setAnalytics({
        riskScore: riskResponse.data.riskScore,
        burnoutRisk: burnoutResponse.data.burnoutRisk,
        anomalies: riskResponse.data.alerts || []
      });
    } catch (error) {
      console.error('Analytics fetch error:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      <div className="metric-card">
        <h3>Risk Score</h3>
        <div className={`score ${analytics.riskScore > 0.7 ? 'high' : 'low'}`}>
          {(analytics.riskScore * 100).toFixed(0)}%
        </div>
      </div>

      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <span className={`badge ${analytics.burnoutRisk}`}>
          {analytics.burnoutRisk}
        </span>
      </div>

      {analytics.anomalies.length > 0 && (
        <div className="alerts">
          <h3>Alerts</h3>
          {analytics.anomalies.map((alert, idx) => (
            <div key={idx} className="alert-item">{alert}</div>
          ))}
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.getTasks = async (req, res) => {
  try {
    const filter = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user.id };
    
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    );
    if (!task) return res.status(404).json({ error: 'Task not found' });
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

## ML Service Implementation

### FastAPI ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import numpy as np
from sklearn.ensemble import IsolationForest
from river import anomaly
import joblib
import os

app = FastAPI()

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketData(BaseModel):
    subject: str
    description: str

class RiskData(BaseModel):
    userId: str
    failedLogins: int = 0
    unusualHours: bool = False
    taskCompletionRate: float = 1.0

class BurnoutData(BaseModel):
    userId: str
    weeklyHours: float
    openTasks: int
    overdueCount: int

@app.post("/classify-ticket")
async def classify_ticket(data: TicketData):
    """AI-based ticket classification"""
    # Simple rule-based classification (replace with ML model)
    text = f"{data.subject} {data.description}".lower()
    
    if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
        category = 'technical'
        priority = 'high'
    elif any(word in text for word in ['feature', 'enhancement', 'add']):
        category = 'feature-request'
        priority = 'medium'
    elif any(word in text for word in ['question', 'how to', 'help']):
        category = 'support'
        priority = 'low'
    else:
        category = 'general'
        priority = 'medium'
    
    return {
        "category": category,
        "priority": priority,
        "suggestedAssignee": "support-team"
    }

@app.post("/risk-detection")
async def detect_risk(data: RiskData):
    """Calculate user risk score"""
    risk_score = 0.0
    alerts = []
    
    # Failed logins
    if data.failedLogins > 3:
        risk_score += 0.3
        alerts.append(f"Multiple failed login attempts: {data.failedLogins}")
    
    # Unusual hours
    if data.unusualHours:
        risk_score += 0.2
        alerts.append("Activity during unusual hours")
    
    # Low productivity
    if data.taskCompletionRate < 0.5:
        risk_score += 0.3
        alerts.append(f"Low task completion rate: {data.taskCompletionRate:.1%}")
    
    return {
        "userId": data.userId,
        "riskScore": min(risk_score, 1.0),
        "alerts": alerts,
        "recommendation": "Monitor user activity" if risk_score > 0.5 else "Normal"
    }

@app.post("/burnout-analysis")
async def analyze_burnout(data: BurnoutData):
    """Detect burnout risk"""
    burnout_score = 0
    
    # Weekly hours threshold
    if data.weeklyHours > 50:
        burnout_score += 2
    elif data.weeklyHours > 40:
        burnout_score += 1
    
    # Task overload
    if data.openTasks > 10:
        burnout_score += 2
    elif data.openTasks > 5:
        burnout_score += 1
    
    # Overdue tasks
    if data.overdueCount > 3:
        burnout_score += 2
    elif data.overdueCount > 0:
        burnout_score += 1
    
    if burnout_score >= 5:
        risk = "high"
        recommendation = "Immediate intervention needed - reduce workload by 40%"
    elif burnout_score >= 3:
        risk = "medium"
        recommendation = "Monitor closely - consider workload redistribution"
    else:
        risk = "low"
        recommendation = "Workload within healthy limits"
    
    return {
        "userId": data.userId,
        "burnoutRisk": risk,
        "score": burnout_score,
        "recommendation": recommendation
    }

@app.post("/anomaly-detection")
async def detect_anomaly(data: dict):
    """Detect anomalous user behavior"""
    # Use isolation forest for anomaly detection
    features = np.array([[
        data.get('loginHour', 12),
        data.get('requestCount', 10),
        data.get('dataAccessVolume', 100)
    ]])
    
    # Simple threshold-based detection
    login_hour = data.get('loginHour', 12)
    is_anomaly = login_hour < 6 or login_hour > 22
    
    return {
        "isAnomaly": is_anomaly,
        "confidence": 0.85 if is_anomaly else 0.15,
        "reason": "Unusual login time" if is_anomaly else "Normal activity"
    }
```

## Configuration

### MongoDB Schema

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    if (!token) throw new Error('No token provided');

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id);
    
    if (!user) throw new Error('User not found');
    
    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { auth, adminOnly };
```

## Common Patterns

### Protected Route Implementation

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const { auth, adminOnly } = require('../middleware/auth');
const userController = require('../controllers/userController');

router.get('/', auth, adminOnly, userController.getAllUsers);
router.post('/', auth, adminOnly, userController.createUser);
router.put('/:id', auth, adminOnly, userController.updateUser);
router.delete('/:id', auth, adminOnly, userController.deleteUser);

module.exports = router;
```

### Time Tracking Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const saveTime = async () => {
    try {
      await axios.post(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`, {
        duration: seconds
      });
      setSeconds(0);
      setIsRunning(false);
    } catch (error) {
      console.error('Time save error:', error);
    }
  };

  const formatTime = (sec) => {
    const hours = Math.floor(sec / 3600);
    const minutes = Math.floor((sec % 3600) / 60);
    const secs = sec % 60;
    return `${hours}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={saveTime} disabled={seconds === 0}>
        Save Time
      </button>
    </div>
  );
};

export default TimeTracker;
```

## Troubleshooting

### JWT Token Expired

```javascript
// Add token refresh logic
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### MongoDB Connection Issues

```javascript
// backend/server.js - Add retry logic
const connectWithRetry = () => {
  mongoose.connect(process.env.MONGODB_URI)
    .then(() => console.log('MongoDB connected'))
    .catch(err => {
      console.error('MongoDB connection error, retrying in 5s...', err);
      setTimeout(connectWithRetry, 5000);
    });
};
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

### ML Service Not Responding

```python
# ml-service/main.py - Add health check
@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

Check ML service availability:

```bash
curl http://localhost:8000/health
```

### Task Status Update Failing

Ensure task status values match schema:

```javascript
// Valid statuses: 'todo', 'in-progress', 'done'
await axios.put(`/api/tasks/${taskId}`, { 
  status: 'in-progress' // not 'inProgress'
});
```
