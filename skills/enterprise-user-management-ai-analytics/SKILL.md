---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with AI insights"
  - "build task management with burnout detection"
  - "integrate AI ticket classification system"
  - "deploy user management system with ML service"
  - "configure JWT authentication for enterprise app"
  - "implement anomaly detection for user behavior"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user/task management with AI-powered insights including risk detection, anomaly detection, burnout analysis, and predictive analytics. Built with React frontend, Node.js backend, and FastAPI ML service using MongoDB.

**Key Capabilities:**
- User authentication and role-based access control (JWT)
- Task management with Kanban boards and time tracking
- Support ticket system with AI-based classification
- AI-powered risk prediction and anomaly detection
- Burnout detection through workload analysis
- Predictive project insights and delay forecasting

## Installation

### Prerequisites

```bash
# Node.js 14+, Python 3.8+, MongoDB
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
```

Create `.env` file:

```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Runs on http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# Runs on http://localhost:8000
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
# Runs on http://localhost:3000
```

## Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   React     │─────▶│  Node.js    │─────▶│  MongoDB    │
│  Frontend   │      │  Backend    │      │  Database   │
└─────────────┘      └─────────────┘      └─────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │   FastAPI   │
                     │ ML Service  │
                     └─────────────┘
```

## Backend API Reference

### Authentication

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
      role: userData.role // 'admin' or 'user'
    })
  });
  return await response.json();
};

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
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

```javascript
// GET /api/users - Get all users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return await response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
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

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### Task Management

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'in_progress', 'done'
    })
  });
  return await response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status })
  });
  return await response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return await response.json();
};

// GET /api/tickets - Get all tickets (Admin)
const getAllTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

## ML Service API Reference

### Risk Prediction

```javascript
// POST /api/ml/predict-risk
const predictUserRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.userId,
      taskCompletionRate: userData.taskCompletionRate,
      avgResponseTime: userData.avgResponseTime,
      loginFrequency: userData.loginFrequency,
      missedDeadlines: userData.missedDeadlines,
      workloadScore: userData.workloadScore
    })
  });
  const result = await response.json();
  // Returns: { riskScore: 0.75, riskLevel: 'high', factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: activityData.userId,
      loginTime: activityData.loginTime,
      loginLocation: activityData.loginLocation,
      actionsPerMinute: activityData.actionsPerMinute,
      unusualPatterns: activityData.unusualPatterns
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.85, type: 'unusual_access' }
  return result;
};
```

### Burnout Detection

```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (userMetrics) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userMetrics.userId,
      workHoursPerWeek: userMetrics.workHoursPerWeek,
      tasksCompleted: userMetrics.tasksCompleted,
      tasksOverdue: userMetrics.tasksOverdue,
      avgTaskDuration: userMetrics.avgTaskDuration,
      consecutiveWorkDays: userMetrics.consecutiveWorkDays
    })
  });
  const result = await response.json();
  // Returns: { burnoutRisk: 'high', score: 0.82, recommendations: [...] }
  return result;
};
```

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      subject: ticketText.subject,
      description: ticketText.description
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', suggestedAssignee: 'user123' }
  return result;
};
```

### Predictive Project Insights

```javascript
// POST /api/ml/predict-project-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-project-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.projectId,
      totalTasks: projectData.totalTasks,
      completedTasks: projectData.completedTasks,
      avgTaskCompletionTime: projectData.avgTaskCompletionTime,
      dueDate: projectData.dueDate,
      teamSize: projectData.teamSize
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.65, estimatedDelay: 5, recommendations: [...] }
  return result;
};
```

## Frontend Component Examples

### React Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect } from 'react';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      // Verify token and get user data
      fetch(`${process.env.REACT_APP_API_URL}/api/auth/verify`, {
        headers: { 'Authorization': `Bearer ${token}` }
      })
        .then(res => res.json())
        .then(data => setUser(data.user))
        .catch(() => {
          localStorage.removeItem('token');
          setToken(null);
        });
    }
  }, [token]);

  const login = async (credentials) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials)
    });
    const data = await response.json();
    
    if (data.token) {
      localStorage.setItem('token', data.token);
      setToken(data.token);
      setUser(data.user);
    }
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return { user, token, login, logout, isAuthenticated: !!token };
};
```

### Task Kanban Board Component

```javascript
// components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const TaskBoard = () => {
  const { token } = useAuth();
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      in_progress: data.filter(t => t.status === 'in_progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
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
      {['todo', 'in_progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status} 
                onChange={(e) => updateTaskStatus(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in_progress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default TaskBoard;
```

### AI Insights Dashboard Component

```javascript
// components/AIInsights.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const AIInsights = ({ userId }) => {
  const { token } = useAuth();
  const [insights, setInsights] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    setLoading(true);
    
    // Fetch user metrics
    const metricsRes = await fetch(
      `${process.env.REACT_APP_API_URL}/api/users/${userId}/metrics`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const metrics = await metricsRes.json();

    // Get AI predictions
    const [riskRes, burnoutRes, anomalyRes] = await Promise.all([
      fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/predict-risk`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(metrics)
      }),
      fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/detect-burnout`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(metrics)
      }),
      fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/detect-anomaly`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(metrics)
      })
    ]);

    const insightsData = {
      risk: await riskRes.json(),
      burnout: await burnoutRes.json(),
      anomaly: await anomalyRes.json()
    };

    setInsights(insightsData);
    setLoading(false);
  };

  if (loading) return <div>Loading AI insights...</div>;

  return (
    <div className="ai-insights-dashboard">
      <div className="insight-card risk">
        <h3>Risk Assessment</h3>
        <p>Risk Level: {insights.risk.riskLevel}</p>
        <p>Score: {insights.risk.riskScore.toFixed(2)}</p>
        <ul>
          {insights.risk.factors.map((factor, i) => (
            <li key={i}>{factor}</li>
          ))}
        </ul>
      </div>

      <div className="insight-card burnout">
        <h3>Burnout Detection</h3>
        <p>Risk: {insights.burnout.burnoutRisk}</p>
        <p>Score: {insights.burnout.score.toFixed(2)}</p>
        <ul>
          {insights.burnout.recommendations.map((rec, i) => (
            <li key={i}>{rec}</li>
          ))}
        </ul>
      </div>

      <div className="insight-card anomaly">
        <h3>Anomaly Detection</h3>
        <p>Status: {insights.anomaly.isAnomaly ? 'Anomaly Detected' : 'Normal'}</p>
        {insights.anomaly.isAnomaly && (
          <>
            <p>Type: {insights.anomaly.type}</p>
            <p>Score: {insights.anomaly.anomalyScore.toFixed(2)}</p>
          </>
        )}
      </div>
    </div>
  );
};

export default AIInsights;
```

## Backend Implementation Patterns

### JWT Middleware

```javascript
// middleware/auth.js
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

### Task Controller Example

```javascript
// controllers/taskController.js
const Task = require('../models/Task');

const createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.user.userId,
      status: 'todo'
    });

    await task.save();
    
    // Trigger AI classification
    const mlResponse = await fetch(`${process.env.ML_SERVICE_URL}/api/ml/classify-task`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title, description })
    });
    const mlData = await mlResponse.json();
    
    task.aiCategory = mlData.category;
    task.estimatedDifficulty = mlData.difficulty;
    await task.save();

    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

const getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

const updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

module.exports = { createTask, getUserTasks, updateTaskStatus };
```

### MongoDB Models

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  isActive: { type: Boolean, default: true },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
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
  completedAt: Date,
  timeTracked: { type: Number, default: 0 }, // in minutes
  aiCategory: String,
  estimatedDifficulty: Number,
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
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
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from river import stream, anomaly
import joblib
import os

app = FastAPI(title="Enterprise ML Service")

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

class UserMetrics(BaseModel):
    userId: str
    taskCompletionRate: float
    avgResponseTime: float
    loginFrequency: int
    missedDeadlines: int
    workloadScore: float

class RiskPrediction(BaseModel):
    riskScore: float
    riskLevel: str
    factors: List[str]

@app.post("/api/ml/predict-risk", response_model=RiskPrediction)
async def predict_risk(metrics: UserMetrics):
    try:
        # Calculate risk score based on multiple factors
        risk_score = 0.0
        factors = []
        
        # Task completion rate factor
        if metrics.taskCompletionRate < 0.7:
            risk_score += 0.3
            factors.append("Low task completion rate")
        
        # Response time factor
        if metrics.avgResponseTime > 24:  # hours
            risk_score += 0.2
            factors.append("Slow response time")
        
        # Login frequency factor
        if metrics.loginFrequency < 3:  # per week
            risk_score += 0.15
            factors.append("Low engagement")
        
        # Missed deadlines factor
        if metrics.missedDeadlines > 3:
            risk_score += 0.25
            factors.append("Multiple missed deadlines")
        
        # Workload factor
        if metrics.workloadScore > 0.8:
            risk_score += 0.1
            factors.append("High workload")
        
        risk_level = "low"
        if risk_score > 0.6:
            risk_level = "high"
        elif risk_score > 0.3:
            risk_level = "medium"
        
        return RiskPrediction(
            riskScore=min(risk_score, 1.0),
            riskLevel=risk_level,
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class ActivityData(BaseModel):
    userId: str
    loginTime: str
    loginLocation: str
    actionsPerMinute: float
    unusualPatterns: List[str]

class AnomalyResult(BaseModel):
    isAnomaly: bool
    anomalyScore: float
    type: Optional[str]

@app.post("/api/ml/detect-anomaly", response_model=AnomalyResult)
async def detect_anomaly(activity: ActivityData):
    try:
        anomaly_score = 0.0
        is_anomaly = False
        anomaly_type = None
        
        # Check for unusual access patterns
        if activity.actionsPerMinute > 10:
            anomaly_score += 0.4
            is_anomaly = True
            anomaly_type = "high_activity_rate"
        
        if len(activity.unusualPatterns) > 0:
            anomaly_score += 0.3 * len(activity.unusualPatterns)
            is_anomaly = True
            anomaly_type = "unusual_access_pattern"
        
        # Time-based anomaly (simplified)
        hour = int(activity.loginTime.split(':')[0])
        if hour < 6 or hour > 22:
            anomaly_score += 0.3
            is_anomaly = True
            anomaly_type = "unusual_time"
        
        return AnomalyResult(
            isAnomaly=is_anomaly,
            anomalyScore=min(anomaly_score, 1.0),
            type=anomaly_type
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class BurnoutMetrics(BaseModel):
    userId: str
    workHoursPerWeek: float
    tasksCompleted: int
    tasksOverdue: int
    avgTaskDuration: float
    consecutiveWorkDays: int

class BurnoutResult(BaseModel):
    burnoutRisk: str
    score: float
    recommendations: List[str]

@app.post("/api/ml/detect-burnout", response_model=BurnoutResult)
async def detect_burnout(metrics: BurnoutMetrics):
    try:
        burnout_score = 0.0
        recommendations = []
        
        # Work hours check
        if metrics.workHoursPerWeek > 50:
            burnout_score += 0.3
            recommendations.append("Consider reducing work hours")
        
        # Overdue tasks stress
        if metrics.tasksOverdue > 5:
            burnout_score += 0.2
            recommendations.append("Reassign overdue tasks")
        
        # Continuous work without breaks
        if metrics.consecutiveWorkDays > 10:
            burnout_score += 0.25
            recommendations.append("Schedule time off")
        
        # Task completion efficiency
        if metrics.avgTaskDuration > 8:  # hours
            burnout_score += 0.15
            recommendations.append("Break tasks into smaller chunks")
        
        # High task load
        if metrics.tasksCompleted > 30:
            burnout_score += 0.1
            recommendations.append("Distribute workload
