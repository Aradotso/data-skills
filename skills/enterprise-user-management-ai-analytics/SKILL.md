---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics for user behavior monitoring
  - implement risk detection and anomaly detection for users
  - create a user management dashboard with AI insights
  - set up JWT authentication with role-based access control
  - build a task management system with burnout detection
  - configure AI-powered ticket classification
  - deploy enterprise user management with FastAPI ML service
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with machine learning capabilities. It provides:

- **User Management**: JWT-based authentication, role-based access control (RBAC), user CRUD operations
- **Task Management**: Kanban boards, time tracking, task assignment and monitoring
- **Support System**: Ticket creation, AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, predictive project insights
- **Admin Dashboard**: Organization analytics, audit logs, performance metrics

The system uses React.js frontend, Node.js/Express backend, MongoDB database, and FastAPI for ML services.

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB
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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=info
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Architecture

```
Frontend (React:3000) → Backend (Node.js:5000) → MongoDB
                              ↓
                        ML Service (FastAPI:8000)
```

## Backend API Reference

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}

// Response
{
  "token": "jwt_token_here",
  "user": {
    "id": "user_id",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user"
  }
}
```

### User Management Endpoints

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { "Authorization": "Bearer ${JWT_TOKEN}" }

// Get user by ID
GET /api/users/:id
Headers: { "Authorization": "Bearer ${JWT_TOKEN}" }

// Update user
PUT /api/users/:id
Headers: { "Authorization": "Bearer ${JWT_TOKEN}" }
{
  "name": "John Updated",
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
Headers: { "Authorization": "Bearer ${JWT_TOKEN}" }
```

### Task Management Endpoints

```javascript
// Create task
POST /api/tasks
Headers: { "Authorization": "Bearer ${JWT_TOKEN}" }
{
  "title": "Implement user authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Get user tasks
GET /api/tasks/user/:userId
Headers: { "Authorization": "Bearer ${JWT_TOKEN}" }

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress" // or "done"
}

// Track time
POST /api/tasks/:id/time
{
  "timeSpent": 3600 // seconds
}
```

### Support Ticket Endpoints

```javascript
// Create ticket
POST /api/tickets
Headers: { "Authorization": "Bearer ${JWT_TOKEN}" }
{
  "title": "Login issue",
  "description": "Cannot login with correct credentials",
  "priority": "high",
  "category": "technical"
}

// Get tickets
GET /api/tickets
Query params: ?status=open&priority=high

// Update ticket
PUT /api/tickets/:id
{
  "status": "resolved",
  "response": "Issue fixed by resetting password"
}
```

## ML Service API Reference

### AI Analytics Endpoints

```python
# Risk Detection
POST http://localhost:8000/api/ml/risk-detection
Content-Type: application/json

{
  "user_id": "user123",
  "login_attempts": 5,
  "failed_logins": 3,
  "access_time": "2026-04-15T23:45:00",
  "ip_changes": 2,
  "suspicious_actions": 1
}

# Response
{
  "risk_score": 0.78,
  "risk_level": "high",
  "factors": ["unusual_access_time", "multiple_failed_logins"],
  "recommendation": "Require additional authentication"
}
```

```python
# Anomaly Detection
POST http://localhost:8000/api/ml/anomaly-detection
{
  "user_id": "user123",
  "activity_pattern": [10, 15, 12, 8, 45, 9], // tasks per day
  "login_locations": ["US", "US", "US", "RU"],
  "data_access_volume": 15000
}

# Response
{
  "is_anomaly": true,
  "anomaly_score": 0.85,
  "anomalies_detected": ["unusual_activity_spike", "suspicious_location"],
  "severity": "critical"
}
```

```python
# Burnout Detection
POST http://localhost:8000/api/ml/burnout-detection
{
  "user_id": "user123",
  "avg_work_hours": 12,
  "tasks_completed": 45,
  "tasks_overdue": 8,
  "weekend_work": true,
  "missed_deadlines": 5
}

# Response
{
  "burnout_risk": "high",
  "burnout_score": 0.82,
  "indicators": ["excessive_hours", "missed_deadlines", "weekend_work"],
  "suggestion": "Redistribute workload and schedule time off"
}
```

```python
# Project Delay Prediction
POST http://localhost:8000/api/ml/predict-delay
{
  "project_id": "proj123",
  "tasks_total": 50,
  "tasks_completed": 20,
  "days_elapsed": 30,
  "days_remaining": 20,
  "team_size": 5,
  "complexity": "high"
}

# Response
{
  "delay_probability": 0.72,
  "expected_delay_days": 15,
  "risk_factors": ["slow_progress", "high_complexity", "insufficient_time"],
  "recommendations": ["add_resources", "reduce_scope"]
}
```

```python
# AI Ticket Classification
POST http://localhost:8000/api/ml/classify-ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when trying to view admin panel",
  "user_role": "user"
}

# Response
{
  "category": "technical",
  "priority": "high",
  "assigned_department": "IT Support",
  "estimated_resolution_time": "2 hours",
  "similar_tickets": ["ticket_456", "ticket_789"]
}
```

## Frontend Integration Patterns

### Authentication with JWT

```javascript
// src/utils/auth.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const authService = {
  async login(email, password) {
    const response = await axios.post(`${API_URL}/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    
    return response.data;
  },
  
  logout() {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  },
  
  getCurrentUser() {
    return JSON.parse(localStorage.getItem('user'));
  },
  
  getToken() {
    return localStorage.getItem('token');
  }
};

// Axios interceptor for auth
axios.interceptors.request.use(
  config => {
    const token = authService.getToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => Promise.reject(error)
);
```

### User Dashboard Component

```javascript
// src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    try {
      const user = JSON.parse(localStorage.getItem('user'));
      
      // Fetch tasks
      const tasksRes = await axios.get(`${API_URL}/tasks/user/${user.id}`);
      const categorizedTasks = {
        todo: tasksRes.data.filter(t => t.status === 'todo'),
        inProgress: tasksRes.data.filter(t => t.status === 'in-progress'),
        done: tasksRes.data.filter(t => t.status === 'done')
      };
      setTasks(categorizedTasks);
      
      // Fetch AI analytics
      const analyticsRes = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`,
        {
          user_id: user.id,
          avg_work_hours: calculateAvgHours(tasksRes.data),
          tasks_completed: categorizedTasks.done.length,
          tasks_overdue: tasksRes.data.filter(t => new Date(t.dueDate) < new Date()).length
        }
      );
      setAnalytics(analyticsRes.data);
      
      setLoading(false);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
      setLoading(false);
    }
  };

  const calculateAvgHours = (tasks) => {
    const totalSeconds = tasks.reduce((sum, task) => sum + (task.timeSpent || 0), 0);
    return totalSeconds / 3600 / tasks.length;
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchUserData(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      {/* Burnout Alert */}
      {analytics?.burnout_risk === 'high' && (
        <div className="alert alert-warning">
          ⚠️ High burnout risk detected: {analytics.suggestion}
        </div>
      )}
      
      {/* Kanban Board */}
      <div className="kanban-board">
        <div className="column">
          <h3>To Do ({tasks.todo.length})</h3>
          {tasks.todo.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => moveTask(task._id, 'in-progress')}>
                Start →
              </button>
            </div>
          ))}
        </div>
        
        <div className="column">
          <h3>In Progress ({tasks.inProgress.length})</h3>
          {tasks.inProgress.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <button onClick={() => moveTask(task._id, 'done')}>
                Complete ✓
              </button>
            </div>
          ))}
        </div>
        
        <div className="column">
          <h3>Done ({tasks.done.length})</h3>
          {tasks.done.map(task => (
            <div key={task._id} className="task-card completed">
              <h4>{task.title}</h4>
              <span>✓ Completed</span>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Dashboard

```javascript
// src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskUsers, setRiskUsers] = useState([]);
  const [anomalies, setAnomalies] = useState([]);

  useEffect(() => {
    fetchAdminData();
  }, []);

  const fetchAdminData = async () => {
    try {
      // Fetch all users
      const usersRes = await axios.get(`${API_URL}/users`);
      setUsers(usersRes.data);
      
      // Run AI analytics for each user
      const riskAnalysis = await Promise.all(
        usersRes.data.map(async (user) => {
          try {
            const riskRes = await axios.post(`${ML_API_URL}/api/ml/risk-detection`, {
              user_id: user._id,
              login_attempts: user.loginAttempts || 0,
              failed_logins: user.failedLogins || 0,
              access_time: new Date().toISOString(),
              ip_changes: user.ipChanges || 0
            });
            return { user, risk: riskRes.data };
          } catch (error) {
            return { user, risk: null };
          }
        })
      );
      
      const highRisk = riskAnalysis.filter(u => u.risk?.risk_level === 'high');
      setRiskUsers(highRisk);
      
    } catch (error) {
      console.error('Error fetching admin data:', error);
    }
  };

  const suspendUser = async (userId) => {
    try {
      await axios.put(`${API_URL}/users/${userId}`, {
        status: 'suspended'
      });
      fetchAdminData(); // Refresh
    } catch (error) {
      console.error('Error suspending user:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      {/* Risk Alerts */}
      <section className="risk-section">
        <h2>⚠️ High Risk Users ({riskUsers.length})</h2>
        {riskUsers.map(({ user, risk }) => (
          <div key={user._id} className="alert-card">
            <h3>{user.name}</h3>
            <p>Risk Score: {(risk.risk_score * 100).toFixed(0)}%</p>
            <p>Factors: {risk.factors.join(', ')}</p>
            <p>Recommendation: {risk.recommendation}</p>
            <button onClick={() => suspendUser(user._id)}>
              Suspend User
            </button>
          </div>
        ))}
      </section>
      
      {/* User Management Table */}
      <section className="users-section">
        <h2>All Users ({users.length})</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.name}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.status || 'active'}</td>
                <td>
                  <button>Edit</button>
                  <button onClick={() => suspendUser(user._id)}>Suspend</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>
    </div>
  );
};

export default AdminDashboard;
```

## Backend Implementation Patterns

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      throw new Error('No token provided');
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');
    
    if (!user) {
      throw new Error('User not found');
    }
    
    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
  }
};

const adminAuth = async (req, res, next) => {
  await auth(req, res, () => {
    if (req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Admin access required' });
    }
    next();
  });
};

module.exports = { auth, adminAuth };
```

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
    lowercase: true
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
  status: {
    type: String,
    enum: ['active', 'suspended', 'inactive'],
    default: 'active'
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
  ipAddresses: [String]
}, {
  timestamps: true
});

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

### Task Model and Routes

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
  timeSpent: {
    type: Number,
    default: 0 // in seconds
  },
  tags: [String]
}, {
  timestamps: true
});

module.exports = mongoose.model('Task', taskSchema);

// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { auth, adminAuth } = require('../middleware/auth');

// Create task
router.post('/', auth, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get user tasks
router.get('/user/:userId', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Track time
router.post('/:id/time', auth, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    task.timeSpent += req.body.timeSpent;
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import IsolationForest, RandomForestClassifier
from river import anomaly, drift
import joblib
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

# Models storage
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Initialize models
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)
risk_model = None  # Will be loaded or trained

class RiskDetectionRequest(BaseModel):
    user_id: str
    login_attempts: int
    failed_logins: int
    access_time: str
    ip_changes: int
    suspicious_actions: int = 0

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    activity_pattern: List[float]
    login_locations: List[str]
    data_access_volume: float

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    avg_work_hours: float
    tasks_completed: int
    tasks_overdue: int
    weekend_work: bool
    missed_deadlines: int

class ProjectDelayRequest(BaseModel):
    project_id: str
    tasks_total: int
    tasks_completed: int
    days_elapsed: int
    days_remaining: int
    team_size: int
    complexity: str

@app.post("/api/ml/risk-detection")
async def detect_risk(request: RiskDetectionRequest):
    try:
        # Calculate risk score based on multiple factors
        risk_score = 0.0
        factors = []
        
        # Failed login ratio
        if request.login_attempts > 0:
            fail_ratio = request.failed_logins / request.login_attempts
            if fail_ratio > 0.3:
                risk_score += 0.3
                factors.append("high_failed_login_ratio")
        
        # Unusual access time (late night)
        from datetime import datetime
        access_hour = datetime.fromisoformat(request.access_time.replace('Z', '')).hour
        if access_hour < 6 or access_hour > 22:
            risk_score += 0.2
            factors.append("unusual_access_time")
        
        # Multiple IP changes
        if request.ip_changes > 3:
            risk_score += 0.25
            factors.append("multiple_ip_changes")
        
        # Suspicious actions
        if request.suspicious_actions > 0:
            risk_score += 0.25 * min(request.suspicious_actions, 1)
            factors.append("suspicious_actions")
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = "high"
            recommendation = "Require additional authentication and review activity"
        elif risk_score >= 0.4:
            risk_level = "medium"
            recommendation = "Monitor user activity closely"
        else:
            risk_level = "low"
            recommendation = "Normal activity"
        
        return {
            "risk_score": round(risk_score, 2),
            "risk_level": risk_level,
            "factors": factors,
            "recommendation": recommendation
        }
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        # Activity pattern analysis
        activity = np.array(request.activity_pattern)
        mean_activity = activity.mean()
        std_activity = activity.std()
        
        anomalies_detected = []
        
        # Check for sudden spikes
        if any(activity > mean_activity + 3 * std_activity):
            anomalies_detected.append("unusual_activity_spike")
        
        # Location check
        unique_locations = len(set(request.login_locations))
        if unique_locations > 3:
            anomalies_detected.append("suspicious_location")
        
        # Data access volume
        if request.data_access_volume > 10000:
            anomalies_detected.append("high_data_access")
        
        # Use River's HalfSpaceTrees for online anomaly detection
        features = {
            'avg_activity': float(mean_activity),
            'activity_variance': float(std_activity),
            'unique_locations': float(unique_locations),
            'data_volume': float(request.data_access_volume)
        }
        
        anomaly_score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = len(anomalies_detected) > 0 or anomaly_score > 0.7
        
        severity = "critical" if len(anomalies_detected) >= 2 else "warning" if is_anomaly else "normal"
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": round(float(anomaly_score), 2),
            "anomalies_detected": anomalies_detected,
            "severity": severity
        }
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        burnout_score = 0.0
        indicators = []
        
        # Work hours analysis
        if request.avg_work_hours > 10:
            burnout_score += 0.3
            indicators.append("excessive_hours")
        
        # Task completion vs overdue ratio
        if request.tasks_completed > 0:
            overdue_ratio = request.tasks_overdue / (request.tasks_completed + request.tasks_overdue)
            if overdue_ratio > 0.2:
                burnout_score += 0.25
                indicators.append("high_overdue_ratio")
        
        # Weekend work
        if request.weekend_work:
            burnout_score += 0.15
            indicators.append("weekend_work")
        
        # Missed deadlines
        if request.missed_deadlines > 3:
            burnout_score += 0.3
            indicators.append("missed_deadlines")
        
        # Determine burnout risk
        if burnout_score >= 0.7:
            burnout_risk = "high"
            suggestion = "Redistribute workload, schedule time off, and review task assignments"
        elif burnout_score >= 0.4:
            burnout_risk = "medium"
            suggestion = "Monitor workload and consider task redistribution"
        else:
            burnout_risk = "low"
            suggestion = "Workload appears manageable"
        
