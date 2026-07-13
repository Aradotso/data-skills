---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and intelligent ticket routing
triggers:
  - "set up enterprise user management with AI analytics"
  - "build a user management system with AI insights"
  - "implement AI-powered ticket classification and routing"
  - "create admin dashboard with user analytics"
  - "add risk detection and burnout analysis to user system"
  - "integrate AI analytics into user management app"
  - "build Kanban task board with time tracking"
  - "implement JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack JavaScript application that combines traditional user and task management with AI-powered analytics. It provides role-based access control (Admin/User), Kanban task boards, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

**Key Components:**
- **Frontend**: React.js SPA with admin and user dashboards
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI with scikit-learn and River for online learning
- **Database**: MongoDB for user, task, and ticket data

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+ and pip
- MongoDB instance (local or cloud)

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt
```

### Environment Configuration

**Backend** (`backend/.env`):
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend** (`frontend/.env`):
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

**ML Service** (`ml-service/.env`):
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the System

### Start Backend
```bash
cd backend
npm start
# Backend runs at http://localhost:5000
```

### Start ML Service
```bash
cd ml-service
uvicorn main:app --reload
# ML service runs at http://localhost:8000
```

### Start Frontend
```bash
cd frontend
npm start
# Frontend runs at http://localhost:3000
```

## Backend API Reference

### Authentication

**Register User**
```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: 'user' // or 'admin'
    })
  });
  return response.json();
};
```

**Login**
```javascript
// POST /api/auth/login
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store token for subsequent requests
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

**Get All Users**
```javascript
// GET /api/users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

**Update User**
```javascript
// PUT /api/users/:id
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

**Delete User**
```javascript
// DELETE /api/users/:id
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management

**Create Task**
```javascript
// POST /api/tasks
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: 'medium', // low, medium, high
      status: 'todo', // todo, in-progress, done
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:id/status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status: newStatus }) // todo, in-progress, done
  });
  return response.json();
};
```

**Get User Tasks**
```javascript
// GET /api/tasks/user/:userId
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Support Tickets

**Create Ticket**
```javascript
// POST /api/tickets
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: 'medium',
      category: 'technical' // technical, billing, general
    })
  });
  return response.json();
};
```

**Get All Tickets (Admin)**
```javascript
// GET /api/tickets
const getAllTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## ML Service API Reference

### AI Ticket Classification

**Classify and Route Ticket**
```javascript
// POST /ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: "Login issues"
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', suggested_assignee: 'team_id' }
  return result;
};
```

### Risk Prediction

**Predict User Risk Score**
```javascript
// POST /ml/predict-risk
const predictUserRisk = async (userId, behaviorData) => {
  const response = await fetch('http://localhost:8000/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      login_frequency: behaviorData.loginFrequency,
      failed_login_attempts: behaviorData.failedLogins,
      task_completion_rate: behaviorData.completionRate,
      avg_task_delay: behaviorData.avgDelay
    })
  });
  const result = await response.json();
  // Returns: { risk_score: 0.75, risk_level: 'high', factors: [...] }
  return result;
};
```

### Anomaly Detection

**Detect Behavioral Anomalies**
```javascript
// POST /ml/detect-anomaly
const detectAnomaly = async (userActivity) => {
  const response = await fetch('http://localhost:8000/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userActivity.userId,
      login_time: userActivity.timestamp,
      ip_address: userActivity.ipAddress,
      location: userActivity.location,
      device: userActivity.device
    })
  });
  const result = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.85, reason: 'unusual_location' }
  return result;
};
```

### Burnout Analysis

**Analyze User Burnout Risk**
```javascript
// POST /ml/burnout-analysis
const analyzeBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/ml/burnout-analysis', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      tasks_count: 45,
      avg_work_hours: 9.5,
      overtime_hours: 15,
      task_completion_rate: 0.65,
      days_without_break: 12
    })
  });
  const result = await response.json();
  // Returns: { burnout_score: 0.82, risk_level: 'high', recommendations: [...] }
  return result;
};
```

### Predictive Insights

**Predict Project Delays**
```javascript
// POST /ml/predict-delays
const predictProjectDelays = async (projectData) => {
  const response = await fetch('http://localhost:8000/ml/predict-delays', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.projectId,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      team_size: projectData.teamSize,
      avg_task_duration: projectData.avgDuration,
      deadline: projectData.deadline
    })
  });
  const result = await response.json();
  // Returns: { delay_probability: 0.68, expected_delay_days: 5, bottlenecks: [...] }
  return result;
};
```

## Frontend Component Patterns

### Protected Route with JWT

```javascript
// src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('userRole');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && userRole !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchUserTasks();
  }, [userId]);

  const fetchUserTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    // Group tasks by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchUserTasks(); // Refresh board
  };

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <button onClick={() => moveTask(task._id, 'in-progress')}>
              Start
            </button>
          </div>
        ))}
      </div>
      <div className="kanban-column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <button onClick={() => moveTask(task._id, 'done')}>
              Complete
            </button>
          </div>
        ))}
      </div>
      <div className="kanban-column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => (
          <div key={task._id} className="task-card completed">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';

const AIAnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    burnoutAlerts: [],
    anomalies: []
  });

  useEffect(() => {
    fetchAIAnalytics();
  }, []);

  const fetchAIAnalytics = async () => {
    const token = localStorage.getItem('token');
    
    // Fetch risk predictions
    const riskResponse = await fetch(`${process.env.REACT_APP_API_URL}/api/analytics/risk-users`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const riskData = await riskResponse.json();

    // Fetch burnout analysis
    const burnoutResponse = await fetch(`${process.env.REACT_APP_API_URL}/api/analytics/burnout`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const burnoutData = await burnoutResponse.json();

    // Fetch anomalies
    const anomalyResponse = await fetch(`${process.env.REACT_APP_API_URL}/api/analytics/anomalies`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const anomalyData = await anomalyResponse.json();

    setAnalytics({
      riskUsers: riskData,
      burnoutAlerts: burnoutData,
      anomalies: anomalyData
    });
  };

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics Dashboard</h2>
      
      <div className="analytics-section">
        <h3>High Risk Users</h3>
        {analytics.riskUsers.map(user => (
          <div key={user.id} className="alert-card risk">
            <p><strong>{user.name}</strong></p>
            <p>Risk Score: {(user.riskScore * 100).toFixed(0)}%</p>
            <p>Factors: {user.factors.join(', ')}</p>
          </div>
        ))}
      </div>

      <div className="analytics-section">
        <h3>Burnout Alerts</h3>
        {analytics.burnoutAlerts.map(alert => (
          <div key={alert.userId} className="alert-card burnout">
            <p><strong>{alert.userName}</strong></p>
            <p>Burnout Score: {(alert.score * 100).toFixed(0)}%</p>
            <p>Recommendations: {alert.recommendations.join(', ')}</p>
          </div>
        ))}
      </div>

      <div className="analytics-section">
        <h3>Security Anomalies</h3>
        {analytics.anomalies.map((anomaly, idx) => (
          <div key={idx} className="alert-card anomaly">
            <p><strong>{anomaly.userName}</strong></p>
            <p>Detected: {anomaly.reason}</p>
            <p>Time: {new Date(anomaly.timestamp).toLocaleString()}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

### Time Tracking Stopwatch

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else {
      clearInterval(interval);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const saveTimeLog = async () => {
    const token = localStorage.getItem('token');
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time-log`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ duration: seconds })
    });
    setSeconds(0);
    setIsRunning(false);
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <div className="controls">
        <button onClick={() => setIsRunning(!isRunning)}>
          {isRunning ? 'Pause' : 'Start'}
        </button>
        <button onClick={() => setSeconds(0)} disabled={isRunning}>
          Reset
        </button>
        <button onClick={saveTimeLog} disabled={isRunning || seconds === 0}>
          Save Log
        </button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

## Backend Implementation Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ error: 'Access denied. No token provided.' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(400).json({ error: 'Invalid token.' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
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
  activity: {
    lastLogin: Date,
    loginCount: { type: Number, default: 0 },
    failedLoginAttempts: { type: Number, default: 0 }
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
  return bcrypt.compare(candidatePassword, this.password);
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
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
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
  dueDate: Date,
  timeTracking: [{
    startTime: Date,
    duration: Number, // in seconds
    loggedAt: { type: Date, default: Date.now }
  }],
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
});

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Task', taskSchema);
```

### Support Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  ticketNumber: {
    type: String,
    unique: true
  },
  subject: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
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
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'security'],
    default: 'general'
  },
  aiClassification: {
    suggestedCategory: String,
    suggestedPriority: String,
    confidence: Number
  },
  comments: [{
    user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
    text: String,
    createdAt: { type: Date, default: Date.now }
  }],
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

// Generate ticket number
ticketSchema.pre('save', async function(next) {
  if (!this.ticketNumber) {
    const count = await mongoose.model('Ticket').countDocuments();
    this.ticketNumber = `TKT-${(count + 1).toString().padStart(6, '0')}`;
  }
  next();
});

module.exports = mongoose.model('Ticket', ticketSchema);
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
    const { username, email, password, role } = req.body;

    const existingUser = await User.findOne({ $or: [{ email }, { username }] });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }

    const user = new User({ username, email, password, role: role || 'user' });
    await user.save();

    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );

    res.status(201).json({
      message: 'User registered successfully',
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
      user.activity.failedLoginAttempts += 1;
      await user.save();
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Update login activity
    user.activity.lastLogin = new Date();
    user.activity.loginCount += 1;
    user.activity.failedLoginAttempts = 0;
    await user.save();

    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );

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

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
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

class TicketClassificationRequest(BaseModel):
    text: str
    subject: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: float
    failed_login_attempts: int
    task_completion_rate: float
    avg_task_delay: float

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    login_time: str
    ip_address: str
    location: str
    device: str

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_count: int
    avg_work_hours: float
    overtime_hours: float
    task_completion_rate: float
    days_without_break: int

@app.get("/")
async def root():
    return {"message": "Enterprise User Management
