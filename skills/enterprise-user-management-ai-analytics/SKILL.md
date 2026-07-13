---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "configure AI analytics for user management"
  - "implement task management with burnout detection"
  - "create admin dashboard with user analytics"
  - "integrate AI ticket classification system"
  - "build user management system with JWT auth"
  - "add predictive analytics to task system"
  - "deploy enterprise management platform"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System, a full-stack application that combines user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Role-based access control, authentication via JWT, user CRUD operations
- **Task Management**: Kanban boards, time tracking, task assignment, progress monitoring
- **Support Tickets**: Smart ticket creation, AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Centralized monitoring, audit logs, organizational analytics
- **User Dashboard**: Personal task views, performance insights, notifications

**Architecture**: React frontend + Node.js/Express backend + FastAPI ML service + MongoDB database

## Installation & Setup

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB running locally or connection string
```

### Complete Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install and run backend
cd backend
npm install
npm start
# Backend runs at http://localhost:5000

# Install and run ML service (in new terminal)
cd ml-service
pip install -r requirements.txt
uvicorn main:app --reload
# ML service runs at http://localhost:8000

# Install and run frontend (in new terminal)
cd frontend
npm install
npm start
# Frontend runs at http://localhost:3000
```

## Configuration

### Backend Configuration

Create `backend/.env`:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_users
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

### ML Service Configuration

Create `ml-service/.env`:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
CORS_ORIGINS=["http://localhost:3000", "http://localhost:5000"]
```

### Frontend Configuration

Create `frontend/.env`:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

## Backend API Usage

### Authentication APIs

```javascript
// Register new user
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return await response.json();
};

// Login user
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
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};

// Get current user profile
const getUserProfile = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/auth/profile', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

### User Management APIs (Admin)

```javascript
// Get all users (admin only)
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Create user (admin)
const createUser = async (userData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(userData)
  });
  return await response.json();
};

// Update user (admin)
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};

// Delete user (admin)
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### Task Management APIs

```javascript
// Get user tasks
const getUserTasks = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Create task
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return await response.json();
};

// Update task status
const updateTaskStatus = async (taskId, status) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status })
  });
  return await response.json();
};

// Track time on task
const trackTaskTime = async (taskId, timeSpent) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ timeSpent }) // in minutes
  });
  return await response.json();
};
```

### Support Ticket APIs

```javascript
// Create support ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return await response.json();
};

// Get user tickets
const getUserTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets/user', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Update ticket (admin/support)
const updateTicket = async (ticketId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};
```

## ML Service API Usage

### Risk Prediction

```javascript
// Predict user risk score
const predictUserRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/predict/risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.userId,
      loginAttempts: userData.loginAttempts,
      failedLogins: userData.failedLogins,
      lastLoginTime: userData.lastLoginTime,
      activityLevel: userData.activityLevel,
      permissionChanges: userData.permissionChanges
    })
  });
  const result = await response.json();
  // result: { riskScore: 0.75, riskLevel: 'high', factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
const detectAnomalies = async (behaviorData) => {
  const response = await fetch('http://localhost:8000/detect/anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: behaviorData.userId,
      loginTimes: behaviorData.loginTimes,
      actionsPerHour: behaviorData.actionsPerHour,
      dataAccessPatterns: behaviorData.dataAccessPatterns,
      ipAddresses: behaviorData.ipAddresses
    })
  });
  const result = await response.json();
  // result: { isAnomaly: true, anomalyScore: 0.89, details: [...] }
  return result;
};
```

### Burnout Detection

```javascript
// Analyze user burnout risk
const analyzeBurnout = async (workloadData) => {
  const response = await fetch('http://localhost:8000/analyze/burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: workloadData.userId,
      tasksCompleted: workloadData.tasksCompleted,
      averageWorkHours: workloadData.averageWorkHours,
      overtimeHours: workloadData.overtimeHours,
      taskCompletionRate: workloadData.taskCompletionRate,
      stressIndicators: workloadData.stressIndicators
    })
  });
  const result = await response.json();
  // result: { burnoutRisk: 'medium', score: 0.65, recommendations: [...] }
  return result;
};
```

### Ticket Classification

```javascript
// AI-based ticket classification and routing
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/classify/ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketText.title,
      description: ticketText.description
    })
  });
  const result = await response.json();
  // result: { category: 'technical', priority: 'high', assignTo: 'team-A' }
  return result;
};
```

### Project Delay Prediction

```javascript
// Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict/delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.projectId,
      totalTasks: projectData.totalTasks,
      completedTasks: projectData.completedTasks,
      averageTaskTime: projectData.averageTaskTime,
      teamSize: projectData.teamSize,
      complexityScore: projectData.complexityScore,
      deadline: projectData.deadline
    })
  });
  const result = await response.json();
  // result: { delayProbability: 0.72, estimatedDelay: '5 days', factors: [...] }
  return result;
};
```

## Frontend Component Patterns

### Protected Route Component

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

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch('http://localhost:5000/api/tasks', {
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
    await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} onMove={moveTask} />
      <Column title="In Progress" tasks={tasks.inProgress} onMove={moveTask} />
      <Column title="Done" tasks={tasks.done} onMove={moveTask} />
    </div>
  );
};
```

### AI Analytics Dashboard Component

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';

const AIAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    const userId = localStorage.getItem('userId');

    // Fetch user data
    const userResponse = await fetch(`http://localhost:5000/api/users/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const userData = await userResponse.json();

    // Get risk prediction
    const riskResponse = await fetch('http://localhost:8000/predict/risk', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        userId: userId,
        loginAttempts: userData.loginAttempts,
        failedLogins: userData.failedLogins,
        activityLevel: userData.activityLevel
      })
    });
    const riskData = await riskResponse.json();

    // Get burnout analysis
    const burnoutResponse = await fetch('http://localhost:8000/analyze/burnout', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        userId: userId,
        averageWorkHours: userData.averageWorkHours,
        overtimeHours: userData.overtimeHours
      })
    });
    const burnoutData = await burnoutResponse.json();

    setAnalytics({
      riskScore: riskData,
      burnoutRisk: burnoutData,
      anomalies: userData.anomalies || []
    });
  };

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h3>Risk Score</h3>
        <p>{analytics.riskScore?.riskLevel}</p>
        <span>{analytics.riskScore?.riskScore}</span>
      </div>
      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <p>{analytics.burnoutRisk?.burnoutRisk}</p>
        <span>{analytics.burnoutRisk?.score}</span>
      </div>
      <div className="anomalies-list">
        <h3>Recent Anomalies</h3>
        {analytics.anomalies.map(a => (
          <div key={a.id}>{a.description}</div>
        ))}
      </div>
    </div>
  );
};
```

## Backend Middleware Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = decoded.userId;
    req.userRole = decoded.role;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.userRole !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Backend Route Example

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { authMiddleware } = require('../middleware/auth');
const Task = require('../models/Task');

// Get user tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.userId })
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.userId
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.userId },
      { $set: req.body },
      { new: true }
    );
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation Patterns

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")
os.makedirs(MODEL_PATH, exist_ok=True)

class RiskPredictionInput(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    activityLevel: float
    permissionChanges: int

class RiskPredictionOutput(BaseModel):
    riskScore: float
    riskLevel: str
    factors: list

@app.post("/predict/risk", response_model=RiskPredictionOutput)
async def predict_risk(data: RiskPredictionInput):
    try:
        # Calculate risk score based on multiple factors
        risk_factors = []
        score = 0.0
        
        # Failed login analysis
        if data.failedLogins > 3:
            score += 0.3
            risk_factors.append("Multiple failed login attempts")
        
        # Activity level analysis
        if data.activityLevel > 100:
            score += 0.2
            risk_factors.append("Unusually high activity")
        
        # Permission changes analysis
        if data.permissionChanges > 2:
            score += 0.25
            risk_factors.append("Multiple permission changes")
        
        # Normalize score
        score = min(score, 1.0)
        
        # Determine risk level
        if score < 0.3:
            risk_level = "low"
        elif score < 0.6:
            risk_level = "medium"
        else:
            risk_level = "high"
        
        return RiskPredictionOutput(
            riskScore=round(score, 2),
            riskLevel=risk_level,
            factors=risk_factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class BurnoutAnalysisInput(BaseModel):
    userId: str
    tasksCompleted: int
    averageWorkHours: float
    overtimeHours: float
    taskCompletionRate: float

class BurnoutAnalysisOutput(BaseModel):
    burnoutRisk: str
    score: float
    recommendations: list

@app.post("/analyze/burnout", response_model=BurnoutAnalysisOutput)
async def analyze_burnout(data: BurnoutAnalysisInput):
    try:
        score = 0.0
        recommendations = []
        
        # Overtime analysis
        if data.overtimeHours > 10:
            score += 0.4
            recommendations.append("Reduce overtime hours")
        
        # Work hours analysis
        if data.averageWorkHours > 50:
            score += 0.3
            recommendations.append("Balance workload distribution")
        
        # Task completion rate
        if data.taskCompletionRate < 0.7:
            score += 0.2
            recommendations.append("Review task assignments")
        
        score = min(score, 1.0)
        
        if score < 0.3:
            risk = "low"
        elif score < 0.6:
            risk = "medium"
            recommendations.append("Monitor workload closely")
        else:
            risk = "high"
            recommendations.append("Immediate intervention required")
        
        return BurnoutAnalysisOutput(
            burnoutRisk=risk,
            score=round(score, 2),
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class TicketClassificationInput(BaseModel):
    title: str
    description: str

class TicketClassificationOutput(BaseModel):
    category: str
    priority: str
    assignTo: str

@app.post("/classify/ticket", response_model=TicketClassificationOutput)
async def classify_ticket(data: TicketClassificationInput):
    try:
        text = f"{data.title} {data.description}".lower()
        
        # Simple keyword-based classification
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = "technical"
            priority = "high"
            assign_to = "team-tech"
        elif any(word in text for word in ['password', 'login', 'access']):
            category = "security"
            priority = "high"
            assign_to = "team-security"
        elif any(word in text for word in ['help', 'how to', 'question']):
            category = "support"
            priority = "medium"
            assign_to = "team-support"
        else:
            category = "general"
            priority = "low"
            assign_to = "team-support"
        
        return TicketClassificationOutput(
            category=category,
            priority=priority,
            assignTo=assign_to
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Database Models

### MongoDB User Model

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
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'support'],
    default: 'user'
  },
  loginAttempts: {
    type: Number,
    default: 0
  },
  failedLogins: {
    type: Number,
    default: 0
  },
  lastLogin: Date,
  isActive: {
    type: Boolean,
    default: true
  },
  metadata: {
    activityLevel: Number,
    averageWorkHours: Number,
    overtimeHours: Number
  }
}, { timestamps: true });

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### MongoDB Task Model

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
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: Date,
  timeTracked: {
    type: Number,
    default: 0 // in minutes
  },
  tags: [String]
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Complete Authentication Flow

```javascript
// Complete login flow with token storage
const handleLogin = async (email, password) => {
  try {
    // Login request
    const response = await fetch('http://localhost:5000/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });

    if (!response.ok) {
      throw new Error('Login failed');
    }

    const data = await response.json();
    
    // Store auth data
    localStorage.setItem('token', data.token);
    localStorage.setItem('userId', data.userId);
    localStorage.setItem('userRole', data.role);
    
    // Fetch user profile
    const profileResponse = await fetch('http://localhost:5000/api/auth/profile', {
      headers: { 'Authorization': `Bearer ${data.token}` }
    });
    const profile = await profileResponse.json();
    
    // Redirect based on role
    if (profile.role === 'admin') {
      window.location.href = '/admin/dashboard';
    } else {
      window.location.href = '/user/dashboard';
    }
  } catch (error) {
    console.error('Login error:', error);
    alert('Login failed. Please check your credentials.');
  }
};
```

### Task Lifecycle Management

```javascript
// Complete task creation and tracking flow
const manageTaskLifecycle = async () => {
  const token = localStorage.getItem('token');
  
  // 1. Create task
  const createResponse = await fetch('http://localhost:5000/api/tasks',
