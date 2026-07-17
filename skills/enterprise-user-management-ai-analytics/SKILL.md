---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, and task optimization
triggers:
  - "set up enterprise user management with AI analytics"
  - "create a user management dashboard with AI insights"
  - "implement AI-based ticket classification system"
  - "add risk prediction to user management app"
  - "build task management with burnout detection"
  - "integrate AI analytics for enterprise users"
  - "deploy user management system with ML service"
  - "configure JWT authentication for user management"
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application combining user/task management with AI-powered insights. It provides risk detection, anomaly detection, burnout analysis, ticket classification, and predictive project analytics using FastAPI ML services, Node.js backend, and React frontend.

## Installation

### Prerequisites
- Node.js 14+
- Python 3.8+
- MongoDB running locally or remotely

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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_db
JWT_SECRET=your_jwt_secret_key_here
NODE_ENV=development
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
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Architecture

The system follows a microservices architecture:
- **Frontend**: React.js (Port 3000)
- **Backend API**: Node.js/Express (Port 5000)
- **ML Service**: FastAPI (Port 8000)
- **Database**: MongoDB

## Backend API Patterns

### Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization').replace('Bearer ', '');
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
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

module.exports = { authMiddleware, adminOnly };
```

### User Management Routes

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { authMiddleware, adminOnly } = require('../middleware/auth');

// Get all users (Admin only)
router.get('/users', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create user (Admin only)
router.post('/users', authMiddleware, adminOnly, async (req, res) => {
  try {
    const { name, email, role, department } = req.body;
    const user = new User({ name, email, role, department });
    await user.save();
    res.status(201).json(user);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update user
router.put('/users/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    ).select('-password');
    if (!user) return res.status(404).json({ error: 'User not found' });
    res.json(user);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Delete user
router.delete('/users/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) return res.status(404).json({ error: 'User not found' });
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Management Routes

```javascript
// routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Get user tasks
router.get('/tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/tasks', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority, assignedTo, dueDate } = req.body;
    const task = new Task({
      title,
      description,
      priority,
      assignedTo,
      assignedBy: req.user.id,
      dueDate,
      status: 'todo'
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update task status
router.patch('/tasks/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { status, updatedAt: Date.now() },
      { new: true }
    );
    if (!task) return res.status(404).json({ error: 'Task not found' });
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Track time on task
router.post('/tasks/:id/time', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body; // duration in seconds
    const task = await Task.findById(req.params.id);
    if (!task) return res.status(404).json({ error: 'Task not found' });
    
    task.timeTracked = (task.timeTracked || 0) + duration;
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Ticket Routes

```javascript
// routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authMiddleware } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/tickets', authMiddleware, async (req, res) => {
  try {
    const { title, description, category } = req.body;
    
    // Get AI classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      { title, description, category }
    );
    
    const ticket = new Ticket({
      title,
      description,
      category: mlResponse.data.category,
      priority: mlResponse.data.priority,
      createdBy: req.user.id,
      assignedTo: mlResponse.data.suggested_assignee,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get user tickets
router.get('/tickets', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .populate('assignedTo', 'name email')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, cluster
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Models storage
models = {}

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    category: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: int
    failed_logins: int
    data_access_volume: float
    unusual_hours: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_pending: int
    avg_task_duration: float
    overtime_hours: float

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    activity_pattern: List[float]

@app.on_event("startup")
async def load_models():
    """Load or initialize ML models"""
    model_path = os.getenv("MODEL_PATH", "./models")
    os.makedirs(model_path, exist_ok=True)
    
    # Initialize anomaly detector
    models['anomaly_detector'] = anomaly.HalfSpaceTrees(
        n_trees=10,
        height=8,
        window_size=250
    )

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and assign priority"""
    # Simple rule-based classification (can be replaced with trained model)
    text = f"{request.title} {request.description}".lower()
    
    # Priority classification
    if any(word in text for word in ['urgent', 'critical', 'down', 'broken']):
        priority = 'high'
    elif any(word in text for word in ['bug', 'error', 'issue']):
        priority = 'medium'
    else:
        priority = 'low'
    
    # Category refinement
    categories = {
        'technical': ['error', 'bug', 'crash', 'performance'],
        'access': ['login', 'password', 'permission', 'access'],
        'feature': ['request', 'enhancement', 'add', 'new']
    }
    
    detected_category = request.category
    for cat, keywords in categories.items():
        if any(word in text for word in keywords):
            detected_category = cat
            break
    
    return {
        "category": detected_category,
        "priority": priority,
        "suggested_assignee": None,
        "confidence": 0.85
    }

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavior patterns"""
    # Calculate risk score
    risk_score = 0.0
    risk_factors = []
    
    # Failed login attempts
    if request.failed_logins > 3:
        risk_score += 0.3
        risk_factors.append("Multiple failed login attempts")
    
    # Unusual access patterns
    if request.unusual_hours > 5:
        risk_score += 0.2
        risk_factors.append("Access during unusual hours")
    
    # High data access volume
    if request.data_access_volume > 1000:
        risk_score += 0.25
        risk_factors.append("Excessive data access")
    
    # Low login frequency (inactive account)
    if request.login_frequency < 1:
        risk_score += 0.15
        risk_factors.append("Inactive account")
    
    risk_level = "high" if risk_score > 0.6 else "medium" if risk_score > 0.3 else "low"
    
    return {
        "user_id": request.user_id,
        "risk_score": min(risk_score, 1.0),
        "risk_level": risk_level,
        "risk_factors": risk_factors,
        "recommendation": "Investigate immediately" if risk_level == "high" else "Monitor activity"
    }

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze user workload for burnout risk"""
    burnout_score = 0.0
    factors = []
    
    # High pending tasks
    if request.tasks_pending > 10:
        burnout_score += 0.3
        factors.append("High number of pending tasks")
    
    # Long task durations
    if request.avg_task_duration > 8:  # hours
        burnout_score += 0.25
        factors.append("Extended task durations")
    
    # Overtime hours
    if request.overtime_hours > 10:
        burnout_score += 0.35
        factors.append("Excessive overtime")
    
    # Low completion rate
    completion_rate = request.tasks_completed / max(request.tasks_completed + request.tasks_pending, 1)
    if completion_rate < 0.5:
        burnout_score += 0.1
        factors.append("Low task completion rate")
    
    burnout_level = "high" if burnout_score > 0.6 else "medium" if burnout_score > 0.3 else "low"
    
    return {
        "user_id": request.user_id,
        "burnout_score": min(burnout_score, 1.0),
        "burnout_level": burnout_level,
        "factors": factors,
        "recommendation": "Reduce workload and schedule break" if burnout_level == "high" else "Monitor workload"
    }

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    detector = models.get('anomaly_detector')
    
    # Convert activity pattern to features
    features = {f'feature_{i}': val for i, val in enumerate(request.activity_pattern)}
    
    # Update and score
    score = detector.score_one(features)
    detector.learn_one(features)
    
    is_anomaly = score > 0.7
    
    return {
        "user_id": request.user_id,
        "anomaly_score": float(score),
        "is_anomaly": is_anomaly,
        "severity": "high" if score > 0.8 else "medium" if score > 0.6 else "low"
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "models_loaded": len(models)}
```

## Frontend React Patterns

### Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
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
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    setUser(user);
    return user;
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
```

### Task Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`);
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
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

  const handleDragStart = (e, task) => {
    e.dataTransfer.setData('taskId', task._id);
  };

  const handleDrop = (e, status) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, status);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status === 'inProgress' ? 'In Progress' : status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              className={`task-card priority-${task.priority}`}
              draggable
              onDragStart={(e) => handleDragStart(e, task)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className="priority-badge">{task.priority}</span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeTasks: 0,
    openTickets: 0,
    riskAlerts: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const [users, tasks, tickets, risks] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/users`),
        axios.get(`${process.env.REACT_APP_API_URL}/api/tasks/analytics`),
        axios.get(`${process.env.REACT_APP_API_URL}/api/tickets/analytics`),
        axios.get(`${process.env.REACT_APP_API_URL}/api/ai/risk-alerts`)
      ]);

      setAnalytics({
        totalUsers: users.data.length,
        activeTasks: tasks.data.active,
        openTickets: tickets.data.open,
        riskAlerts: risks.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-value">{analytics.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p className="stat-value">{analytics.activeTasks}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p className="stat-value">{analytics.openTickets}</p>
        </div>
      </div>

      <div className="risk-alerts">
        <h2>Risk Alerts</h2>
        {analytics.riskAlerts.map(alert => (
          <div key={alert.user_id} className={`alert alert-${alert.risk_level}`}>
            <h4>User: {alert.user_name}</h4>
            <p>Risk Score: {(alert.risk_score * 100).toFixed(0)}%</p>
            <ul>
              {alert.risk_factors.map((factor, i) => (
                <li key={i}>{factor}</li>
              ))}
            </ul>
            <p><strong>{alert.recommendation}</strong></p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: {
    type: String,
    required: true
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
    enum: ['todo', 'in_progress', 'done'],
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
  timeTracked: {
    type: Number,
    default: 0  // in seconds
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Making AI-Enhanced API Calls

```javascript
// Integrate ML service with backend API
const axios = require('axios');

async function createTicketWithAI(ticketData) {
  try {
    // Get AI classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      {
        title: ticketData.title,
        description: ticketData.description,
        category: ticketData.category
      }
    );
    
    // Create ticket with AI insights
    const ticket = new Ticket({
      ...ticketData,
      category: mlResponse.data.category,
      priority: mlResponse.data.priority,
      aiConfidence: mlResponse.data.confidence
    });
    
    await ticket.save();
    return ticket;
  } catch (error) {
    console.error('AI classification failed:', error);
    // Fallback to manual classification
    const ticket = new Ticket(ticketData);
    await ticket.save();
    return ticket;
  }
}
```

### Scheduled Risk Analysis

```javascript
// backend/jobs/riskAnalysis.js
const cron = require('node-cron');
const axios = require('axios');
const User = require('../models/User');
const AuditLog = require('../models/AuditLog');

// Run risk analysis every 6 hours
cron.schedule('0 */6 * * *', async () => {
  console.log('Running risk analysis...');
  
  const users = await User.find({ isActive: true });
  
  for (const user of users) {
    const logs = await AuditLog.find({ userId: user._id })
      .sort({ createdAt: -1 })
      .limit(100);
    
    const riskData = {
      user_id: user._id.toString(),
      login_frequency: calculateLoginFrequency(logs),
      failed_logins: countFailedLogins(logs),
      data_access_volume: calculateDataAccess(logs),
      unusual_hours: countUnusualHours(logs)
    };
    
    try {
      const response = await axios.post(
        `${process.env.ML_SERVICE_URL}/predict-risk`,
        riskData
      );
      
      if (response.data.risk_level === 'high') {
        // Send alert to admins
        await sendRiskAlert(user, response.data);
      }
    } catch (error) {
      console.error(`Risk analysis failed for user ${user.email}:`, error);
    }
  }
});
```

## Configuration

### Environment Variables

Backend (`.env`):
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_db
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
```

ML Service (`.env`):
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
ANOMALY_THRESHOLD=0.7
BURNOUT_THRESHOLD=0.6
```

Frontend (`.env`):
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
REACT_APP_ENABLE_AI=true
```

## Troubleshooting

### ML Service Connection Issues

**Problem**: Frontend or backend cannot reach ML service

**Solution**:
```javascript
// Add timeout and retry logic
const axios = require('axios');

async function callMLService(endpoint, data, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      const response = await axios.post(
        `${process.env.ML_SERVICE_URL}${endpoint}`,
        data,
        { timeout: 5000 }
      );
      return response.data;
    } catch (error) {
      if (i === retries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }
}
```

### MongoDB Connection Errors

**Problem**: `MongoNetworkError` or connection timeout

**Solution**:
```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### JWT Token Expiration

**Problem**: Users getting logged out frequently

**Solution**:
```javascript
// Implement token refresh mechanism
const refreshToken = async () => {
  try {
    const
