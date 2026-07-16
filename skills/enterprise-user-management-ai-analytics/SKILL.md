---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement task management with risk prediction"
  - "create ticket classification system with ML"
  - "build user dashboard with burnout detection"
  - "configure AI-powered admin analytics"
  - "implement anomaly detection for user behavior"
  - "set up JWT authentication with role-based access"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with machine learning capabilities. It provides:

- **User Management**: JWT authentication, role-based access control (Admin/User)
- **Task Management**: Kanban board, time tracking, task assignment
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, alerts

**Stack**: React frontend, Node.js/Express backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites

```bash
# Required: Node.js 14+, Python 3.8+, MongoDB
node --version
python --version
mongod --version
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

# Create .env file
cat > .env << EOF
PORT=5000
MONGO_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=\${YOUR_JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
DATABASE_URL=mongodb://localhost:27017/enterprise_user_mgmt
LOG_LEVEL=info
EOF

uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
EOF

npm start
```

## Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   React     │─────▶│  Node.js    │─────▶│  MongoDB    │
│  Frontend   │◀─────│  Backend    │◀─────│  Database   │
└─────────────┘      └──────┬──────┘      └─────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │  FastAPI    │
                     │ ML Service  │
                     └─────────────┘
```

## Backend API Reference

### Authentication

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
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = new User({ username, email, password, role });
    await user.save();

    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );

    res.status(201).json({ token, user: { id: user._id, username, email, role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user || !(await user.comparePassword(password))) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );

    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ message: 'No authentication token' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Get user tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority, assignedTo, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      priority,
      assignedTo: assignedTo || req.user.userId,
      assignedBy: req.user.userId,
      dueDate,
      status: 'todo'
    });

    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body; // todo, in-progress, done
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    await task.save();

    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Ticket Management

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const axios = require('axios');
const { authMiddleware } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
      title,
      description
    });

    const { category, suggested_priority, routing_department } = mlResponse.data;

    const ticket = new Ticket({
      title,
      description,
      priority: priority || suggested_priority,
      category,
      department: routing_department,
      createdBy: req.user.userId,
      status: 'open'
    });

    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get tickets
router.get('/', authMiddleware, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.userId };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, preprocessing
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")
os.makedirs(MODEL_PATH, exist_ok=True)

# Anomaly detector (online learning)
anomaly_detector = anomaly.HalfSpaceTrees(seed=42)

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    suggested_priority: str
    routing_department: str
    confidence: float

class RiskPredictionRequest(BaseModel):
    user_id: str
    task_count: int
    overdue_tasks: int
    avg_completion_time: float
    login_frequency: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    avg_task_duration: float
    overtime_hours: float
    stress_indicators: List[float]

@app.get("/")
def root():
    return {"message": "Enterprise User Management ML Service", "status": "running"}

@app.post("/classify-ticket", response_model=TicketClassificationResponse)
def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and suggest routing"""
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple keyword-based classification (replace with trained model)
        if any(word in text for word in ["bug", "error", "crash", "not working"]):
            category = "Technical"
            department = "Engineering"
            priority = "high"
        elif any(word in text for word in ["access", "permission", "login"]):
            category = "Access"
            department = "IT Support"
            priority = "medium"
        elif any(word in text for word in ["feature", "enhancement", "request"]):
            category = "Feature Request"
            department = "Product"
            priority = "low"
        else:
            category = "General"
            department = "Support"
            priority = "medium"
        
        return TicketClassificationResponse(
            category=category,
            suggested_priority=priority,
            routing_department=department,
            confidence=0.85
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavior patterns"""
    try:
        # Calculate risk score
        features = np.array([
            request.task_count,
            request.overdue_tasks,
            request.avg_completion_time,
            request.login_frequency
        ]).reshape(1, -1)
        
        # Simple risk calculation (replace with trained model)
        overdue_ratio = request.overdue_tasks / max(request.task_count, 1)
        risk_score = overdue_ratio * 0.5 + (request.avg_completion_time / 100) * 0.3
        
        if risk_score > 0.7:
            risk_level = "high"
            recommendations = ["Reassign some tasks", "Schedule 1-on-1 meeting"]
        elif risk_score > 0.4:
            risk_level = "medium"
            recommendations = ["Monitor workload", "Check for blockers"]
        else:
            risk_level = "low"
            recommendations = ["Continue current approach"]
        
        return {
            "user_id": request.user_id,
            "risk_score": float(risk_score),
            "risk_level": risk_level,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
def detect_anomaly(data: dict):
    """Detect anomalous user behavior using online learning"""
    try:
        features = {
            "login_time": data.get("login_time", 0),
            "session_duration": data.get("session_duration", 0),
            "actions_count": data.get("actions_count", 0),
            "failed_attempts": data.get("failed_attempts", 0)
        }
        
        # Score the observation
        score = anomaly_detector.score_one(features)
        is_anomaly = score > 0.7
        
        # Learn from this observation
        anomaly_detector.learn_one(features)
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": float(score),
            "user_id": data.get("user_id"),
            "timestamp": data.get("timestamp")
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze employee burnout risk"""
    try:
        # Calculate burnout indicators
        workload_score = min(request.tasks_completed / 50, 1.0)  # Normalized
        overtime_score = min(request.overtime_hours / 20, 1.0)
        stress_score = np.mean(request.stress_indicators) if request.stress_indicators else 0
        
        burnout_score = (workload_score * 0.3 + overtime_score * 0.4 + stress_score * 0.3)
        
        if burnout_score > 0.7:
            level = "critical"
            actions = ["Immediate intervention required", "Reduce workload", "Mandatory time off"]
        elif burnout_score > 0.5:
            level = "warning"
            actions = ["Monitor closely", "Distribute tasks", "Encourage breaks"]
        else:
            level = "healthy"
            actions = ["Continue current pace"]
        
        return {
            "user_id": request.user_id,
            "burnout_score": float(burnout_score),
            "burnout_level": level,
            "recommended_actions": actions
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-project-delay")
def predict_project_delay(data: dict):
    """Predict likelihood of project delay"""
    try:
        completed = data.get("tasks_completed", 0)
        total = data.get("total_tasks", 1)
        days_left = data.get("days_remaining", 1)
        avg_velocity = data.get("avg_daily_velocity", 1)
        
        completion_rate = completed / total
        required_velocity = (total - completed) / max(days_left, 1)
        
        if required_velocity > avg_velocity * 1.5:
            delay_probability = 0.85
            risk = "high"
        elif required_velocity > avg_velocity:
            delay_probability = 0.5
            risk = "medium"
        else:
            delay_probability = 0.15
            risk = "low"
        
        return {
            "project_id": data.get("project_id"),
            "delay_probability": float(delay_probability),
            "risk_level": risk,
            "completion_rate": float(completion_rate),
            "suggested_actions": ["Increase resources", "Adjust timeline"] if risk == "high" else []
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Implementation

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
      loadUser();
    } else {
      setLoading(false);
    }
  }, []);

  const loadUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(res.data);
    } catch (error) {
      console.error('Failed to load user', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', res.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
    setUser(res.data.user);
    return res.data;
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

### Task Kanban Board

```javascript
// frontend/src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './TaskBoard.css';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/tasks`);
      const grouped = {
        todo: res.data.filter(t => t.status === 'todo'),
        inProgress: res.data.filter(t => t.status === 'in-progress'),
        done: res.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task', error);
    }
  };

  const TaskColumn = ({ title, status, taskList }) => (
    <div className="task-column">
      <h3>{title} ({taskList.length})</h3>
      <div className="task-list">
        {taskList.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority ${task.priority}`}>{task.priority}</span>
            <div className="task-actions">
              {status !== 'todo' && (
                <button onClick={() => updateTaskStatus(task._id, 'todo')}>← To Do</button>
              )}
              {status !== 'in-progress' && (
                <button onClick={() => updateTaskStatus(task._id, 'in-progress')}>
                  In Progress
                </button>
              )}
              {status !== 'done' && (
                <button onClick={() => updateTaskStatus(task._id, 'done')}>Done →</button>
              )}
            </div>
          </div>
        ))}
      </div>
    </div>
  );

  return (
    <div className="task-board">
      <TaskColumn title="To Do" status="todo" taskList={tasks.todo} />
      <TaskColumn title="In Progress" status="in-progress" taskList={tasks.inProgress} />
      <TaskColumn title="Done" status="done" taskList={tasks.done} />
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { LineChart, Line, BarChart, Bar, XAxis, YAxis, Tooltip, Legend } from 'recharts';

const AIAnalytics = () => {
  const [riskAnalysis, setRiskAnalysis] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [anomalies, setAnomalies] = useState([]);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch risk prediction
      const riskRes = await axios.post(`${process.env.REACT_APP_ML_URL}/predict-risk`, {
        user_id: 'current_user',
        task_count: 25,
        overdue_tasks: 3,
        avg_completion_time: 4.5,
        login_frequency: 15
      });
      setRiskAnalysis(riskRes.data);

      // Fetch burnout analysis
      const burnoutRes = await axios.post(`${process.env.REACT_APP_ML_URL}/analyze-burnout`, {
        user_id: 'current_user',
        tasks_completed: 42,
        avg_task_duration: 3.2,
        overtime_hours: 12,
        stress_indicators: [0.6, 0.7, 0.8]
      });
      setBurnoutData(burnoutRes.data);
    } catch (error) {
      console.error('Failed to fetch analytics', error);
    }
  };

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Analytics</h2>

      {riskAnalysis && (
        <div className="risk-card">
          <h3>Risk Analysis</h3>
          <div className={`risk-level ${riskAnalysis.risk_level}`}>
            {riskAnalysis.risk_level.toUpperCase()}
          </div>
          <p>Risk Score: {(riskAnalysis.risk_score * 100).toFixed(1)}%</p>
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {riskAnalysis.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        </div>
      )}

      {burnoutData && (
        <div className="burnout-card">
          <h3>Burnout Analysis</h3>
          <div className={`burnout-level ${burnoutData.burnout_level}`}>
            {burnoutData.burnout_level.toUpperCase()}
          </div>
          <p>Burnout Score: {(burnoutData.burnout_score * 100).toFixed(1)}%</p>
          <div className="actions">
            <h4>Recommended Actions:</h4>
            <ul>
              {burnoutData.recommended_actions.map((action, idx) => (
                <li key={idx}>{action}</li>
              ))}
            </ul>
          </div>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

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
  profile: {
    firstName: String,
    lastName: String,
    department: String,
    position: String
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
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
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
  dueDate: Date,
  completedAt: Date,
  timeTracked: {
    type: Number,
    default: 0  // in minutes
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
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
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  category: String,
  department: String,
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  resolution: String,
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### Making Authenticated Requests

```javascript
// Set token on login
const login = async (email, password) => {
  const response = await axios.post('/api/auth/login', { email, password });
  localStorage.setItem('token', response.data.token);
  axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
};

// All subsequent requests automatically include token
const getTasks = async () => {
  const response = await axios.get('/api/tasks');
  return response.data;
};
```

### Integrating ML Predictions

```javascript
// Get AI insights for a user
const getUserInsights = async (userId) => {
  // Fetch user metrics
  const metrics = await axios.get(`/api/analytics/user/${userId}`);
  
  // Get AI predictions
  const [risk, burnout, anomaly] = await Promise.all([
    axios.post(`${process.env.REACT_APP_ML_URL}/predict-risk`, {
      user_id: userId,
      ...metrics.data
    }),
    axios.post(`${process.env.REACT_APP_ML_URL}/analyze-burnout`, {
      user_id: userId,
      ...metrics.data
    }),
    axios.post(`${process.env.REACT_APP_ML_URL}/detect-anomaly`, {
      user_id: userId,
      ...metrics.data
    })
  ]);
  
  return {
    risk: risk.data,
    burnout: burnout.data,
    anomaly: anomaly.data
  };
};
```

### Admin User Management

```javascript
// Admin: Get all users with analytics
const getAllUsersWithAnalytics = async () => {
  const users = await axios.get('/api/admin/users');
  
  // Enrich with AI insights
  const enrichedUsers = await Promise.all(
    users.data.map(async (user) => {
      const insights = await getUserInsights(user._id);
      return { ...user, insights };
    })
  );
  
  return enrichedUsers;
