---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, task tracking, and ticket management
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "build admin panel with role-based access"
  - "add AI ticket classification system"
  - "implement burnout detection analytics"
  - "set up JWT authentication for user management"
  - "create kanban board for task management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with AI-powered insights. It provides role-based access control, task tracking with Kanban boards, support ticket management, and ML-based analytics including risk prediction, anomaly detection, and burnout analysis.

**Stack**: React.js frontend, Node.js/Express backend, MongoDB database, FastAPI ML service with scikit-learn

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB (local or cloud)
mongod --version
```

### Setup

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

**Backend `.env`** (in `backend/` directory):
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend `.env`** (in `frontend/` directory):
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

**ML Service `.env`** (in `ml-service/` directory):
```bash
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the Application

### Start Backend Server

```bash
cd backend
npm start
# Server runs at http://localhost:5000
```

### Start ML Service

```bash
cd ml-service
uvicorn main:app --reload --port 8000
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

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "username": "john.doe",
  "email": "john@company.com",
  "password": "securePassword123",
  "role": "user",
  "department": "Engineering"
}

// Login
POST /api/auth/login
Content-Type: application/json

{
  "email": "john@company.com",
  "password": "securePassword123"
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
// Get all users
GET /api/users
Authorization: Bearer {token}

// Get user by ID
GET /api/users/:id
Authorization: Bearer {token}

// Update user
PUT /api/users/:id
Authorization: Bearer {token}
Content-Type: application/json

{
  "role": "admin",
  "department": "Management",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
Authorization: Bearer {token}
```

### Task Management

```javascript
// Create task
POST /api/tasks
Authorization: Bearer {token}
Content-Type: application/json

{
  "title": "Implement authentication",
  "description": "Add JWT-based authentication",
  "assignedTo": "507f1f77bcf86cd799439011",
  "priority": "high",
  "dueDate": "2026-05-01T00:00:00.000Z",
  "status": "todo"
}

// Get user tasks
GET /api/tasks/user/:userId
Authorization: Bearer {token}

// Update task status
PATCH /api/tasks/:id/status
Authorization: Bearer {token}
Content-Type: application/json

{
  "status": "in-progress"
}

// Track time on task
POST /api/tasks/:id/time
Authorization: Bearer {token}
Content-Type: application/json

{
  "duration": 3600, // seconds
  "action": "start" // or "stop"
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer {token}
Content-Type: application/json

{
  "title": "Cannot access reports",
  "description": "Getting 403 error when accessing reports page",
  "priority": "medium",
  "category": "technical"
}

// Get all tickets (admin)
GET /api/tickets
Authorization: Bearer {token}

// Get user tickets
GET /api/tickets/user/:userId
Authorization: Bearer {token}

// Update ticket
PATCH /api/tickets/:id
Authorization: Bearer {token}
Content-Type: application/json

{
  "status": "in-progress",
  "assignedTo": "507f1f77bcf86cd799439011"
}
```

## ML Service API

### AI Ticket Classification

```python
# POST /classify-ticket
import requests

response = requests.post(
    "http://localhost:8000/classify-ticket",
    json={
        "title": "Database connection timeout",
        "description": "Application cannot connect to database, experiencing timeouts",
        "userId": "507f1f77bcf86cd799439011"
    }
)

result = response.json()
# {
#   "category": "technical",
#   "priority": "high",
#   "suggestedAssignee": "507f1f77bcf86cd799439012",
#   "confidence": 0.87
# }
```

### Risk Prediction

```python
# POST /predict-risk
response = requests.post(
    "http://localhost:8000/predict-risk",
    json={
        "userId": "507f1f77bcf86cd799439011",
        "taskCount": 15,
        "overdueCount": 3,
        "avgCompletionTime": 72, # hours
        "ticketCount": 5
    }
)

result = response.json()
# {
#   "riskLevel": "medium",
#   "riskScore": 0.65,
#   "factors": ["high overdue rate", "increased ticket count"],
#   "recommendations": ["Redistribute tasks", "Check workload"]
# }
```

### Anomaly Detection

```python
# POST /detect-anomaly
response = requests.post(
    "http://localhost:8000/detect-anomaly",
    json={
        "userId": "507f1f77bcf86cd799439011",
        "loginTime": "2026-04-15T02:30:00Z",
        "location": "192.168.1.100",
        "actions": ["delete_user", "modify_permissions"]
    }
)

result = response.json()
# {
#   "isAnomaly": true,
#   "anomalyScore": 0.82,
#   "reason": "Unusual login time and high-risk actions"
# }
```

### Burnout Detection

```python
# POST /detect-burnout
response = requests.post(
    "http://localhost:8000/detect-burnout",
    json={
        "userId": "507f1f77bcf86cd799439011",
        "weeklyHours": 65,
        "tasksCompleted": 12,
        "tasksOverdue": 4,
        "avgTaskTime": 8.5, # hours
        "ticketsRaised": 7
    }
)

result = response.json()
# {
#   "burnoutRisk": "high",
#   "score": 0.78,
#   "indicators": ["excessive hours", "high ticket count"],
#   "suggestions": ["Reduce workload", "Schedule time off"]
# }
```

## Frontend Implementation

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
      fetchUser();
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${API_URL}/auth/me`);
      setUser(response.data);
    } catch (error) {
      logout();
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/auth/login`, {
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

  return { user, token, login, logout, isAuthenticated: !!token };
};
```

### Task Management Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/tasks/user/${userId}`);
      const grouped = groupTasksByStatus(response.data);
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const groupTasksByStatus = (taskList) => {
    return taskList.reduce((acc, task) => {
      const status = task.status === 'in-progress' ? 'inProgress' : task.status;
      if (!acc[status]) acc[status] = [];
      acc[status].push(task);
      return acc;
    }, { todo: [], inProgress: [], done: [] });
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task, onStatusChange }) => (
    <div className="task-card">
      <h3>{task.title}</h3>
      <p>{task.description}</p>
      <span className={`priority-${task.priority}`}>{task.priority}</span>
      <select
        value={task.status}
        onChange={(e) => onStatusChange(task._id, e.target.value)}
      >
        <option value="todo">To Do</option>
        <option value="in-progress">In Progress</option>
        <option value="done">Done</option>
      </select>
    </div>
  );

  return (
    <div className="kanban-board">
      <div className="column">
        <h2>To Do ({tasks.todo.length})</h2>
        {tasks.todo.map(task => (
          <TaskCard
            key={task._id}
            task={task}
            onStatusChange={updateTaskStatus}
          />
        ))}
      </div>
      <div className="column">
        <h2>In Progress ({tasks.inProgress.length})</h2>
        {tasks.inProgress.map(task => (
          <TaskCard
            key={task._id}
            task={task}
            onStatusChange={updateTaskStatus}
          />
        ))}
      </div>
      <div className="column">
        <h2>Done ({tasks.done.length})</h2>
        {tasks.done.map(task => (
          <TaskCard
            key={task._id}
            task={task}
            onStatusChange={updateTaskStatus}
          />
        ))}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_URL = process.env.REACT_APP_ML_URL;

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    burnoutUsers: [],
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const [riskResponse, burnoutResponse, anomalyResponse] = await Promise.all([
        axios.get(`${ML_URL}/analytics/risk`),
        axios.get(`${ML_URL}/analytics/burnout`),
        axios.get(`${ML_URL}/analytics/anomalies`)
      ]);

      setAnalytics({
        riskUsers: riskResponse.data,
        burnoutUsers: burnoutResponse.data,
        anomalies: anomalyResponse.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="analytics-dashboard">
      <section className="risk-section">
        <h2>High Risk Users</h2>
        {analytics.riskUsers.map(user => (
          <div key={user.userId} className="alert-card risk">
            <h3>{user.username}</h3>
            <p>Risk Score: {(user.riskScore * 100).toFixed(0)}%</p>
            <ul>
              {user.factors.map((factor, i) => (
                <li key={i}>{factor}</li>
              ))}
            </ul>
          </div>
        ))}
      </section>

      <section className="burnout-section">
        <h2>Burnout Risk</h2>
        {analytics.burnoutUsers.map(user => (
          <div key={user.userId} className="alert-card burnout">
            <h3>{user.username}</h3>
            <p>Burnout Risk: {user.burnoutRisk}</p>
            <p>Weekly Hours: {user.weeklyHours}</p>
            <ul>
              {user.suggestions.map((suggestion, i) => (
                <li key={i}>{suggestion}</li>
              ))}
            </ul>
          </div>
        ))}
      </section>

      <section className="anomaly-section">
        <h2>Security Anomalies</h2>
        {analytics.anomalies.map((anomaly, index) => (
          <div key={index} className="alert-card anomaly">
            <h3>{anomaly.username}</h3>
            <p>Time: {new Date(anomaly.timestamp).toLocaleString()}</p>
            <p>Reason: {anomaly.reason}</p>
            <p>Score: {(anomaly.anomalyScore * 100).toFixed(0)}%</p>
          </div>
        ))}
      </section>
    </div>
  );
};

export default AdminAnalytics;
```

## Backend Implementation

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
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization?.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Invalid token' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        message: 'User role not authorized'
      });
    }
    next();
  };
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user._id
    });
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({
      assignedTo: req.params.userId
    }).populate('createdBy', 'username email');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status },
      { new: true, runValidators: true }
    );

    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    if (req.body.action === 'start') {
      task.timeTracking = {
        startTime: new Date(),
        isActive: true
      };
    } else if (req.body.action === 'stop') {
      const duration = (new Date() - task.timeTracking.startTime) / 1000;
      task.totalTimeSpent = (task.totalTimeSpent || 0) + duration;
      task.timeTracking.isActive = false;
    }

    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};
```

## ML Service Implementation

### Main FastAPI Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Optional
import os

app = FastAPI(title="Enterprise AI Analytics")

# Load or initialize models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    userId: str

class RiskPredictionRequest(BaseModel):
    userId: str
    taskCount: int
    overdueCount: int
    avgCompletionTime: float
    ticketCount: int

class BurnoutDetectionRequest(BaseModel):
    userId: str
    weeklyHours: float
    tasksCompleted: int
    tasksOverdue: int
    avgTaskTime: float
    ticketsRaised: int

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify ticket using ML model"""
    try:
        # Feature extraction
        text_length = len(request.title + request.description)
        has_urgent = "urgent" in request.description.lower()
        has_error = "error" in request.description.lower()
        
        # Simple rule-based classification (replace with actual ML model)
        if has_urgent or has_error:
            priority = "high"
        elif text_length > 200:
            priority = "medium"
        else:
            priority = "low"
        
        # Determine category
        tech_keywords = ["database", "error", "crash", "bug", "timeout"]
        category = "technical" if any(kw in request.description.lower() for kw in tech_keywords) else "general"
        
        return {
            "category": category,
            "priority": priority,
            "suggestedAssignee": None,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk level"""
    try:
        # Calculate risk score
        overdue_ratio = request.overdueCount / max(request.taskCount, 1)
        ticket_factor = min(request.ticketCount / 10, 1.0)
        time_factor = min(request.avgCompletionTime / 100, 1.0)
        
        risk_score = (overdue_ratio * 0.5 + ticket_factor * 0.3 + time_factor * 0.2)
        
        if risk_score > 0.7:
            risk_level = "high"
            factors = ["high overdue rate", "excessive tickets"]
        elif risk_score > 0.4:
            risk_level = "medium"
            factors = ["moderate overdue rate"]
        else:
            risk_level = "low"
            factors = []
        
        recommendations = []
        if risk_level in ["high", "medium"]:
            recommendations = ["Redistribute tasks", "Check workload", "Schedule review"]
        
        return {
            "riskLevel": risk_level,
            "riskScore": risk_score,
            "factors": factors,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    """Detect burnout risk"""
    try:
        # Calculate burnout score
        hours_factor = min(request.weeklyHours / 80, 1.0)
        overdue_ratio = request.tasksOverdue / max(request.tasksCompleted + request.tasksOverdue, 1)
        ticket_factor = min(request.ticketsRaised / 10, 1.0)
        
        burnout_score = (hours_factor * 0.4 + overdue_ratio * 0.3 + ticket_factor * 0.3)
        
        if burnout_score > 0.7:
            risk = "high"
            indicators = ["excessive hours", "high ticket count", "poor completion rate"]
            suggestions = ["Reduce workload", "Schedule time off", "Redistribute tasks"]
        elif burnout_score > 0.4:
            risk = "medium"
            indicators = ["elevated hours", "moderate stress indicators"]
            suggestions = ["Monitor workload", "Consider break periods"]
        else:
            risk = "low"
            indicators = []
            suggestions = []
        
        return {
            "burnoutRisk": risk,
            "score": burnout_score,
            "indicators": indicators,
            "suggestions": suggestions
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

## Common Patterns

### Protected Route Component

```javascript
// src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, requiredRole }) => {
  const { user, isAuthenticated } = useAuth();

  if (!isAuthenticated) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Axios Interceptor Setup

```javascript
// src/utils/axios.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// Request interceptor
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

## Troubleshooting

### JWT Token Issues

```javascript
// Verify token in backend
const jwt = require('jsonwebtoken');

try {
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  console.log('Token valid:', decoded);
} catch (error) {
  console.error('Token error:', error.message);
  // Common errors: TokenExpiredError, JsonWebTokenError
}
```

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Configuration

```javascript
// backend/server.js
const cors = require('cors');
const express = require('express');

const app = express();

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Not Responding

```python
# Check ML service health
import requests

try:
    response = requests.get('http://localhost:8000/health', timeout=5)
    print(f"ML Service Status: {response.json()}")
except requests.exceptions.RequestException as e:
    print(f"ML Service Error: {e}")
```

### Task Time Tracking Issues

```javascript
// Ensure proper state management for timer
const [isTracking, setIsTracking] = useState(false);
const [startTime, setStartTime] = useState(null);
const [elapsedTime, setElapsedTime] = useState(0);

useEffect(() => {
  let interval;
  if (isTracking && startTime) {
    interval = setInterval(() => {
      setElapsedTime(Date.now() - startTime);
    }, 1000);
  }
  return () => clearInterval(interval);
}, [isTracking, startTime]);
```

### Environment Variables Not Loading

```bash
# Ensure .env file is in correct directory
# Restart development server after .env changes
# Verify React env vars start with REACT_APP_

# Check if loaded
console.log('API URL:', process.env.REACT_APP_API_URL);
```
