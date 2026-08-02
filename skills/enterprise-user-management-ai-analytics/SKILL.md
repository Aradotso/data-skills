---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement user task tracking with AI insights"
  - "create admin dashboard for user management"
  - "integrate AI-based ticket classification system"
  - "build kanban board with time tracking"
  - "add burnout detection and risk prediction"
  - "configure JWT authentication for user management"
  - "deploy enterprise management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack JavaScript application that combines user/task management with AI-powered analytics. It features role-based access control (admin/user), Kanban-style task tracking, support ticket management, and ML-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

**Architecture:**
- **Frontend:** React.js (port 3000)
- **Backend:** Node.js with Express (port 5000)
- **ML Service:** FastAPI with scikit-learn/River (port 8000)
- **Database:** MongoDB
- **Auth:** JWT-based authentication

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ for ML service
python --version

# MongoDB installed and running
mongod --version
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Setup backend
cd backend
npm install
cp .env.example .env  # Configure environment variables

# Setup ML service
cd ../ml-service
pip install -r requirements.txt

# Setup frontend
cd ../frontend
npm install
```

### Environment Configuration

**backend/.env:**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_management
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**frontend/.env:**
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

**ml-service/.env:**
```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the Application

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload --port 8000

# Terminal 3 - Frontend
cd frontend
npm start
```

Access the application at `http://localhost:3000`

## Backend API Reference

### Authentication Endpoints

```javascript
// User Registration
POST /api/auth/register
Content-Type: application/json

{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "SecurePass123",
  "role": "user"  // or "admin"
}

// User Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "SecurePass123"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "507f1f77bcf86cd799439011",
    "username": "john.doe",
    "role": "user"
  }
}
```

### User Management (Admin Only)

```javascript
// Get All Users
GET /api/users
Authorization: Bearer {token}

// Create User
POST /api/users
Authorization: Bearer {token}
{
  "username": "jane.smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update User
PUT /api/users/:userId
Authorization: Bearer {token}
{
  "role": "admin",
  "department": "Management"
}

// Delete User
DELETE /api/users/:userId
Authorization: Bearer {token}
```

### Task Management

```javascript
// Get User Tasks
GET /api/tasks?userId={userId}&status={status}
Authorization: Bearer {token}

// Create Task
POST /api/tasks
Authorization: Bearer {token}
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "507f1f77bcf86cd799439011",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update Task Status
PUT /api/tasks/:taskId
Authorization: Bearer {token}
{
  "status": "in-progress",
  "timeSpent": 3600  // seconds
}

// Track Time
POST /api/tasks/:taskId/time
Authorization: Bearer {token}
{
  "duration": 7200,  // 2 hours in seconds
  "date": "2026-04-15"
}
```

### Support Tickets

```javascript
// Create Ticket
POST /api/tickets
Authorization: Bearer {token}
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when accessing admin dashboard",
  "priority": "high",
  "category": "technical"
}

// Get Tickets
GET /api/tickets?status={status}&assignedTo={userId}
Authorization: Bearer {token}

// Update Ticket
PUT /api/tickets/:ticketId
Authorization: Bearer {token}
{
  "status": "in-progress",
  "assignedTo": "507f1f77bcf86cd799439011",
  "response": "Investigating permission issues"
}
```

## ML Service API Reference

### AI Ticket Classification

```python
# POST /classify-ticket
import requests

ticket_data = {
    "title": "Cannot login to system",
    "description": "Getting authentication error after password reset",
    "user_history": {
        "previous_tickets": 3,
        "avg_resolution_time": 24
    }
}

response = requests.post(
    "http://localhost:8000/classify-ticket",
    json=ticket_data
)

# Response
{
    "category": "authentication",
    "priority": "high",
    "suggested_assignee": "tech_support_team",
    "estimated_resolution_hours": 4
}
```

### Risk Prediction

```python
# POST /predict-risk
user_behavior = {
    "user_id": "507f1f77bcf86cd799439011",
    "login_frequency": 15,
    "failed_login_attempts": 3,
    "unusual_access_times": True,
    "data_access_volume": 1500,
    "recent_role_change": False
}

response = requests.post(
    "http://localhost:8000/predict-risk",
    json=user_behavior
)

# Response
{
    "risk_score": 0.78,
    "risk_level": "high",
    "factors": [
        "Failed login attempts above threshold",
        "Unusual access times detected"
    ],
    "recommended_actions": [
        "Require password reset",
        "Enable 2FA",
        "Monitor user activity"
    ]
}
```

### Anomaly Detection

```python
# POST /detect-anomaly
activity_data = {
    "user_id": "507f1f77bcf86cd799439011",
    "timestamp": "2026-04-15T17:20:35Z",
    "actions": [
        {"type": "data_export", "count": 50},
        {"type": "access_sensitive", "count": 20},
        {"type": "permission_change", "count": 5}
    ],
    "baseline_period_days": 30
}

response = requests.post(
    "http://localhost:8000/detect-anomaly",
    json=activity_data
)

# Response
{
    "is_anomaly": True,
    "anomaly_score": 0.92,
    "deviations": {
        "data_export": "250% above baseline",
        "access_sensitive": "180% above baseline"
    }
}
```

### Burnout Detection

```python
# POST /detect-burnout
workload_data = {
    "user_id": "507f1f77bcf86cd799439011",
    "hours_worked_week": 65,
    "tasks_completed": 45,
    "tasks_overdue": 8,
    "weekend_work_hours": 12,
    "consecutive_work_days": 14,
    "avg_task_completion_time_hours": 6.5
}

response = requests.post(
    "http://localhost:8000/detect-burnout",
    json=workload_data
)

# Response
{
    "burnout_risk": "high",
    "burnout_score": 0.85,
    "indicators": [
        "Excessive weekly hours (65h vs 40h standard)",
        "Working 14 consecutive days",
        "High overdue task ratio"
    ],
    "recommendations": [
        "Redistribute workload",
        "Mandate time off",
        "Review task assignments"
    ]
}
```

### Predictive Project Insights

```python
# POST /predict-project-delay
project_data = {
    "project_id": "proj_12345",
    "total_tasks": 50,
    "completed_tasks": 15,
    "days_elapsed": 30,
    "total_duration_days": 90,
    "team_size": 5,
    "avg_task_complexity": 7,
    "blockers": 3
}

response = requests.post(
    "http://localhost:8000/predict-project-delay",
    json=project_data
)

# Response
{
    "delay_predicted": True,
    "estimated_delay_days": 15,
    "completion_probability": 0.65,
    "risk_factors": [
        "Current completion rate below target",
        "Multiple blockers present"
    ],
    "mitigation_strategies": [
        "Add 2 team members",
        "Resolve blockers priority",
        "Extend deadline by 15 days"
    ]
}
```

## Frontend Integration Patterns

### React Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchCurrentUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchCurrentUser = async () => {
    try {
      const { data } = await axios.get(`${API_URL}/auth/me`);
      setUser(data.user);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const { data } = await axios.post(`${API_URL}/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${data.token}`;
    setUser(data.user);
    return data.user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return { user, loading, login, logout, isAdmin: user?.role === 'admin' };
};
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const { data } = await axios.get(`${API_URL}/tasks?userId=${userId}`);
    
    const categorized = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    
    setTasks(categorized);
  };

  const handleDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');

    if (currentStatus === newStatus) return;

    await axios.put(`${API_URL}/tasks/${taskId}`, { status: newStatus });
    fetchTasks();
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const renderColumn = (status, title, taskList) => (
    <div
      className="kanban-column"
      onDrop={(e) => handleDrop(e, status)}
      onDragOver={handleDragOver}
    >
      <h3>{title} ({taskList.length})</h3>
      {taskList.map(task => (
        <div
          key={task._id}
          className="task-card"
          draggable
          onDragStart={(e) => handleDragStart(e, task._id, status)}
        >
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority-${task.priority}`}>
            {task.priority}
          </span>
          <div className="task-meta">
            Due: {new Date(task.dueDate).toLocaleDateString()}
          </div>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do', tasks.todo)}
      {renderColumn('in-progress', 'In Progress', tasks.inProgress)}
      {renderColumn('done', 'Done', tasks.done)}
    </div>
  );
};

export default KanbanBoard;
```

### AI Insights Dashboard Component

```javascript
// components/AIInsightsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_URL = process.env.REACT_APP_ML_URL;
const API_URL = process.env.REACT_APP_API_URL;

const AIInsightsDashboard = ({ userId }) => {
  const [insights, setInsights] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    try {
      // Get user behavior data
      const { data: userData } = await axios.get(
        `${API_URL}/users/${userId}/behavior`
      );

      // Get risk prediction
      const riskResponse = await axios.post(`${ML_URL}/predict-risk`, {
        user_id: userId,
        login_frequency: userData.loginFrequency,
        failed_login_attempts: userData.failedLogins,
        unusual_access_times: userData.unusualAccess,
        data_access_volume: userData.dataAccess,
        recent_role_change: userData.roleChanged
      });

      // Get burnout detection
      const burnoutResponse = await axios.post(`${ML_URL}/detect-burnout`, {
        user_id: userId,
        hours_worked_week: userData.hoursWorked,
        tasks_completed: userData.tasksCompleted,
        tasks_overdue: userData.tasksOverdue,
        weekend_work_hours: userData.weekendHours,
        consecutive_work_days: userData.consecutiveDays,
        avg_task_completion_time_hours: userData.avgCompletionTime
      });

      // Get anomalies
      const anomalyResponse = await axios.post(`${ML_URL}/detect-anomaly`, {
        user_id: userId,
        timestamp: new Date().toISOString(),
        actions: userData.recentActions,
        baseline_period_days: 30
      });

      setInsights({
        riskScore: riskResponse.data,
        burnoutRisk: burnoutResponse.data,
        anomalies: anomalyResponse.data.is_anomaly 
          ? [anomalyResponse.data] 
          : []
      });
    } catch (error) {
      console.error('Failed to fetch AI insights:', error);
    }
  };

  return (
    <div className="ai-insights-dashboard">
      <h2>AI-Powered Insights</h2>
      
      {/* Risk Score */}
      <div className="insight-card">
        <h3>Security Risk Assessment</h3>
        {insights.riskScore && (
          <>
            <div className={`risk-level-${insights.riskScore.risk_level}`}>
              Risk Level: {insights.riskScore.risk_level.toUpperCase()}
              <span className="risk-score">
                {(insights.riskScore.risk_score * 100).toFixed(0)}%
              </span>
            </div>
            <ul>
              {insights.riskScore.factors.map((factor, i) => (
                <li key={i}>{factor}</li>
              ))}
            </ul>
          </>
        )}
      </div>

      {/* Burnout Risk */}
      <div className="insight-card">
        <h3>Burnout Detection</h3>
        {insights.burnoutRisk && (
          <>
            <div className={`burnout-${insights.burnoutRisk.burnout_risk}`}>
              {insights.burnoutRisk.burnout_risk.toUpperCase()} RISK
              <span className="burnout-score">
                {(insights.burnoutRisk.burnout_score * 100).toFixed(0)}%
              </span>
            </div>
            <h4>Indicators:</h4>
            <ul>
              {insights.burnoutRisk.indicators.map((indicator, i) => (
                <li key={i}>{indicator}</li>
              ))}
            </ul>
            <h4>Recommendations:</h4>
            <ul>
              {insights.burnoutRisk.recommendations.map((rec, i) => (
                <li key={i}>{rec}</li>
              ))}
            </ul>
          </>
        )}
      </div>

      {/* Anomalies */}
      {insights.anomalies.length > 0 && (
        <div className="insight-card alert">
          <h3>⚠️ Anomalies Detected</h3>
          {insights.anomalies.map((anomaly, i) => (
            <div key={i} className="anomaly-item">
              <strong>Score: {(anomaly.anomaly_score * 100).toFixed(0)}%</strong>
              <ul>
                {Object.entries(anomaly.deviations).map(([key, value]) => (
                  <li key={key}>{key}: {value}</li>
                ))}
              </ul>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

export default AIInsightsDashboard;
```

## Common Patterns

### Protected Route with Role Check

```javascript
// components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  
  if (!user) return <Navigate to="/login" />;
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import ProtectedRoute from './components/ProtectedRoute';
import AdminDashboard from './pages/AdminDashboard';
import UserDashboard from './pages/UserDashboard';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route 
          path="/admin" 
          element={
            <ProtectedRoute adminOnly>
              <AdminDashboard />
            </ProtectedRoute>
          } 
        />
        <Route 
          path="/dashboard" 
          element={
            <ProtectedRoute>
              <UserDashboard />
            </ProtectedRoute>
          } 
        />
      </Routes>
    </BrowserRouter>
  );
}
```

### Time Tracking Component

```javascript
// components/TimeTracker.jsx
import React, { useState, useEffect, useRef } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setElapsedTime(prev => prev + 1);
      }, 1000);
    } else {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    }

    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, [isRunning]);

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const handleStart = () => {
    setIsRunning(true);
  };

  const handlePause = () => {
    setIsRunning(false);
  };

  const handleStop = async () => {
    setIsRunning(false);
    
    if (elapsedTime > 0) {
      await axios.post(`${API_URL}/tasks/${taskId}/time`, {
        duration: elapsedTime,
        date: new Date().toISOString()
      });
    }
    
    setElapsedTime(0);
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(elapsedTime)}</div>
      <div className="controls">
        {!isRunning ? (
          <button onClick={handleStart}>Start</button>
        ) : (
          <button onClick={handlePause}>Pause</button>
        )}
        <button onClick={handleStop} disabled={elapsedTime === 0}>
          Stop & Save
        </button>
      </div>
    </div>
  );
};

export default TimeTracker;
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

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### User Model with MongoDB

```javascript
// models/User.js
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
    minlength: 8
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  loginAttempts: {
    type: Number,
    default: 0
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
  
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.user.id,
      status: 'todo'
    });

    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const { userId, status } = req.query;
    
    const query = { assignedTo: userId };
    if (status) query.status = status;

    const tasks = await Task.find(query)
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const { taskId } = req.params;
    const updates = req.body;

    const task = await Task.findByIdAndUpdate(
      taskId,
      { ...updates, updatedAt: Date.now() },
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
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

app = FastAPI(title="Enterprise ML Service")

# CORS configuration
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

class TicketData(BaseModel):
    title: str
    description: str
    user_history: Optional[Dict] = {}

class RiskData(BaseModel):
    user_id: str
    login_frequency: int
    failed_login_attempts: int
    unusual_access_times: bool
    data_access_volume: int
    recent_role_change: bool

class BurnoutData(BaseModel):
    user_id: str
    hours_worked_week: float
    tasks_completed: int
    tasks_overdue: int
    weekend_work_hours: float
    consecutive_work_days: int
    avg_task_completion_time_hours: float

@app.post("/classify-ticket")
async def classify_ticket(data: TicketData):
    """Classify support ticket using NLP and ML"""
    
    # Simple keyword-based classification
    text = (data.title + " " + data.description).lower()
    
    if any(word in text for word in ["login", "password", "access", "auth"]):
        category = "authentication"
        priority = "high"
    elif any(word in text for word in ["bug", "error", "crash", "broken"]):
        category = "technical"
        priority = "high"
    elif any(word in text for word in ["feature", "request", "add", "new"]):
        category = "feature_request"
        priority = "medium"
    else:
        category = "general"
        priority = "low"
    
    return {
        "category": category,
        "priority": priority,
        "suggested_assignee": f"{category}_team",
        "estimated_resolution_hours": 4 if priority == "high" else 8
    }

@app.post("/predict-risk")
async def predict_risk(data: RiskData):
    """Predict user security risk"""
    
    risk_score = 0.0
    factors = []
    
    # Failed login attempts
    if data.failed_login_attempts > 3:
        risk_score += 0.3
        factors.append("Failed login attempts above threshold")
    
    # Unusual access times
    if data.unusual_access_times:
        risk_score += 0.2
        factors.append("Unusual access times detected")
    
    # High data access volume
    if data.data_access_volume > 1000:
        risk_
