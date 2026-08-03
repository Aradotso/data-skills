---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user management dashboard with AI insights"
  - "add risk detection to user management"
  - "build admin dashboard with anomaly detection"
  - "integrate AI ticket classification system"
  - "deploy user management system with ML service"
  - "configure enterprise user management with FastAPI ML"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user/task management with AI-powered insights. It provides role-based access control, task tracking with Kanban boards, support ticket management, and ML-driven features including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

**Stack:**
- Frontend: React.js
- Backend: Node.js with Express
- ML Service: FastAPI with scikit-learn and River
- Database: MongoDB
- Auth: JWT

## Installation

### Prerequisites

```bash
# Ensure you have installed:
# - Node.js 14+
# - Python 3.8+
# - MongoDB
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
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
EOF

npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
EOF

npm start
# Frontend runs at http://localhost:3000
```

## Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   React     │─────▶│   Node.js   │─────▶│   MongoDB   │
│   Frontend  │      │   Backend   │      │   Database  │
└─────────────┘      └─────────────┘      └─────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │   FastAPI   │
                     │  ML Service │
                     └─────────────┘
```

## Key API Endpoints

### Authentication

```javascript
// Register user
POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@company.com",
  "password": "securepass123",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securepass123"
}
// Response: { "token": "jwt_token", "user": {...} }
```

### User Management

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { "Authorization": "Bearer ${JWT_TOKEN}" }

// Update user
PUT /api/users/:id
{
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
```

### Task Management

```javascript
// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Add new analytics dashboard",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:id/time
{
  "timeSpent": 120 // minutes
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "medium",
  "category": "technical"
}

// Get AI classification
POST /api/tickets/:id/classify
// ML service automatically categorizes and routes ticket
```

## ML Service API

### Risk Prediction

```python
# POST /predict/risk
import requests

response = requests.post(
    f"{ML_SERVICE_URL}/predict/risk",
    json={
        "user_id": "user123",
        "failed_logins": 3,
        "unusual_activity": 2,
        "task_delay_rate": 0.4,
        "access_violations": 1
    }
)

risk_score = response.json()
# {"risk_level": "medium", "score": 0.65, "factors": [...]}
```

### Anomaly Detection

```python
# POST /detect/anomaly
response = requests.post(
    f"{ML_SERVICE_URL}/detect/anomaly",
    json={
        "user_id": "user123",
        "login_time": "2026-04-15T03:30:00Z",
        "location": "Unknown",
        "device": "new_device",
        "actions": ["bulk_delete", "role_change"]
    }
)

# {"is_anomaly": true, "confidence": 0.87, "alert": true}
```

### Burnout Detection

```python
# POST /predict/burnout
response = requests.post(
    f"{ML_SERVICE_URL}/predict/burnout",
    json={
        "user_id": "user123",
        "tasks_completed": 45,
        "tasks_pending": 23,
        "avg_working_hours": 11.5,
        "missed_deadlines": 8,
        "weekend_work": 6
    }
)

# {"burnout_risk": "high", "score": 0.82, "recommendations": [...]}
```

### Ticket Classification

```python
# POST /classify/ticket
response = requests.post(
    f"{ML_SERVICE_URL}/classify/ticket",
    json={
        "title": "Cannot access reports",
        "description": "Getting 403 error when trying to view analytics",
        "user_role": "manager"
    }
)

# {"category": "access_control", "priority": "high", "department": "IT"}
```

## Frontend Integration Patterns

### Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  const login = async (email, password) => {
    const response = await axios.post(
      `${process.env.REACT_APP_API_URL}/api/auth/login`,
      { email, password }
    );
    
    setToken(response.data.token);
    setUser(response.data.user);
    localStorage.setItem('token', response.data.token);
    
    return response.data;
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
  };

  return { user, token, login, logout };
};
```

### Task Management Component

```javascript
// components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/tasks`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    
    const grouped = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await axios.patch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      { status: newStatus },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} onMove={updateTaskStatus} />
      <Column title="In Progress" tasks={tasks.inProgress} onMove={updateTaskStatus} />
      <Column title="Done" tasks={tasks.done} onMove={updateTaskStatus} />
    </div>
  );
};
```

### AI Analytics Dashboard

```javascript
// components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    anomalies: [],
    burnoutAlerts: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    
    // Get high-risk users
    const riskResponse = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/analytics/risk-users`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    
    // Get recent anomalies
    const anomalyResponse = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/analytics/anomalies`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    
    // Get burnout alerts
    const burnoutResponse = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/analytics/burnout-alerts`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    
    setAnalytics({
      riskUsers: riskResponse.data,
      anomalies: anomalyResponse.data,
      burnoutAlerts: burnoutResponse.data
    });
  };

  return (
    <div className="admin-dashboard">
      <section>
        <h2>High Risk Users</h2>
        {analytics.riskUsers.map(user => (
          <div key={user.id} className="alert-card">
            <span>{user.name}</span>
            <span className="risk-score">Risk: {user.riskScore}</span>
            <button onClick={() => reviewUser(user.id)}>Review</button>
          </div>
        ))}
      </section>
      
      <section>
        <h2>Anomaly Alerts</h2>
        {analytics.anomalies.map(anomaly => (
          <div key={anomaly.id} className="alert-card">
            <span>{anomaly.description}</span>
            <span>{new Date(anomaly.timestamp).toLocaleString()}</span>
          </div>
        ))}
      </section>
      
      <section>
        <h2>Burnout Alerts</h2>
        {analytics.burnoutAlerts.map(alert => (
          <div key={alert.userId} className="alert-card">
            <span>{alert.userName}</span>
            <span>Score: {alert.burnoutScore}</span>
            <ul>
              {alert.recommendations.map((rec, i) => (
                <li key={i}>{rec}</li>
              ))}
            </ul>
          </div>
        ))}
      </section>
    </div>
  );
};
```

## Backend Implementation Patterns

### Express Server Setup

```javascript
// server.js
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
});

// JWT middleware
const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];
  
  if (!token) return res.sendStatus(401);
  
  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) return res.sendStatus(403);
    req.user = user;
    next();
  });
};

// Role-based middleware
const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', authenticateToken, require('./routes/users'));
app.use('/api/tasks', authenticateToken, require('./routes/tasks'));
app.use('/api/tickets', authenticateToken, require('./routes/tickets'));
app.use('/api/analytics', authenticateToken, requireAdmin, require('./routes/analytics'));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### User Model

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive', 'suspended'], default: 'active' },
  profile: {
    firstName: String,
    lastName: String,
    department: String,
    position: String
  },
  metrics: {
    tasksCompleted: { type: Number, default: 0 },
    tasksPending: { type: Number, default: 0 },
    avgWorkingHours: { type: Number, default: 0 },
    loginAttempts: { type: Number, default: 0 },
    lastLogin: Date
  },
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
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
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
  dueDate: Date,
  timeTracked: { type: Number, default: 0 }, // minutes
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Analytics Route

```javascript
// routes/analytics.js
const express = require('express');
const axios = require('axios');
const User = require('../models/User');
const Task = require('../models/Task');
const AuditLog = require('../models/AuditLog');

const router = express.Router();
const ML_SERVICE_URL = process.env.ML_SERVICE_URL;

// Get high-risk users
router.get('/risk-users', async (req, res) => {
  try {
    const users = await User.find({ status: 'active' });
    const riskAssessments = [];
    
    for (const user of users) {
      const tasks = await Task.find({ assignedTo: user._id });
      const auditLogs = await AuditLog.find({ userId: user._id });
      
      const failedLogins = auditLogs.filter(
        log => log.action === 'login_failed'
      ).length;
      
      const delayedTasks = tasks.filter(
        task => task.dueDate && task.dueDate < new Date() && task.status !== 'done'
      ).length;
      
      const mlResponse = await axios.post(`${ML_SERVICE_URL}/predict/risk`, {
        user_id: user._id.toString(),
        failed_logins: failedLogins,
        unusual_activity: auditLogs.filter(log => log.severity === 'high').length,
        task_delay_rate: delayedTasks / (tasks.length || 1),
        access_violations: auditLogs.filter(log => log.action === 'access_denied').length
      });
      
      if (mlResponse.data.risk_level !== 'low') {
        riskAssessments.push({
          id: user._id,
          name: user.username,
          email: user.email,
          riskScore: mlResponse.data.score,
          riskLevel: mlResponse.data.risk_level,
          factors: mlResponse.data.factors
        });
      }
    }
    
    res.json(riskAssessments.sort((a, b) => b.riskScore - a.riskScore));
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get burnout alerts
router.get('/burnout-alerts', async (req, res) => {
  try {
    const users = await User.find({ status: 'active' });
    const burnoutAlerts = [];
    
    for (const user of users) {
      const tasks = await Task.find({ assignedTo: user._id });
      const completed = tasks.filter(t => t.status === 'done');
      const pending = tasks.filter(t => t.status !== 'done');
      const missed = tasks.filter(
        t => t.dueDate && t.dueDate < new Date() && t.status !== 'done'
      );
      
      const mlResponse = await axios.post(`${ML_SERVICE_URL}/predict/burnout`, {
        user_id: user._id.toString(),
        tasks_completed: completed.length,
        tasks_pending: pending.length,
        avg_working_hours: user.metrics.avgWorkingHours || 8,
        missed_deadlines: missed.length,
        weekend_work: 0 // Calculate from audit logs
      });
      
      if (mlResponse.data.burnout_risk !== 'low') {
        burnoutAlerts.push({
          userId: user._id,
          userName: user.username,
          burnoutScore: mlResponse.data.score,
          riskLevel: mlResponse.data.burnout_risk,
          recommendations: mlResponse.data.recommendations
        });
      }
    }
    
    res.json(burnoutAlerts);
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
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
import joblib
import os

app = FastAPI()

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

class RiskPredictionInput(BaseModel):
    user_id: str
    failed_logins: int
    unusual_activity: int
    task_delay_rate: float
    access_violations: int

class AnomalyDetectionInput(BaseModel):
    user_id: str
    login_time: str
    location: str
    device: str
    actions: List[str]

class BurnoutPredictionInput(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_pending: int
    avg_working_hours: float
    missed_deadlines: int
    weekend_work: int

class TicketClassificationInput(BaseModel):
    title: str
    description: str
    user_role: str

@app.post("/predict/risk")
async def predict_risk(data: RiskPredictionInput):
    """Predict user risk level based on behavioral metrics"""
    
    # Calculate risk score
    risk_score = (
        data.failed_logins * 0.25 +
        data.unusual_activity * 0.30 +
        data.task_delay_rate * 0.25 +
        data.access_violations * 0.20
    )
    
    # Normalize to 0-1
    risk_score = min(risk_score / 10, 1.0)
    
    # Determine risk level
    if risk_score < 0.3:
        risk_level = "low"
    elif risk_score < 0.7:
        risk_level = "medium"
    else:
        risk_level = "high"
    
    factors = []
    if data.failed_logins > 2:
        factors.append("Multiple failed login attempts")
    if data.task_delay_rate > 0.3:
        factors.append("High task delay rate")
    if data.access_violations > 0:
        factors.append("Access violations detected")
    
    return {
        "user_id": data.user_id,
        "risk_level": risk_level,
        "score": round(risk_score, 2),
        "factors": factors
    }

@app.post("/detect/anomaly")
async def detect_anomaly(data: AnomalyDetectionInput):
    """Detect anomalous user behavior"""
    
    is_anomaly = False
    confidence = 0.0
    reasons = []
    
    # Check login time (outside business hours)
    from datetime import datetime
    login_hour = datetime.fromisoformat(data.login_time.replace('Z', '+00:00')).hour
    if login_hour < 6 or login_hour > 22:
        is_anomaly = True
        confidence += 0.3
        reasons.append("Login outside business hours")
    
    # Check location
    if data.location.lower() == "unknown":
        is_anomaly = True
        confidence += 0.25
        reasons.append("Unknown location")
    
    # Check device
    if "new" in data.device.lower():
        confidence += 0.2
        reasons.append("New device detected")
    
    # Check suspicious actions
    suspicious_actions = ["bulk_delete", "role_change", "mass_export"]
    if any(action in data.actions for action in suspicious_actions):
        is_anomaly = True
        confidence += 0.25
        reasons.append("Suspicious actions detected")
    
    confidence = min(confidence, 1.0)
    
    return {
        "user_id": data.user_id,
        "is_anomaly": is_anomaly,
        "confidence": round(confidence, 2),
        "alert": confidence > 0.7,
        "reasons": reasons
    }

@app.post("/predict/burnout")
async def predict_burnout(data: BurnoutPredictionInput):
    """Predict employee burnout risk"""
    
    # Calculate burnout score
    workload_factor = (data.tasks_pending / max(data.tasks_completed, 1)) * 0.3
    hours_factor = max((data.avg_working_hours - 8) / 8, 0) * 0.3
    deadline_factor = (data.missed_deadlines / max(data.tasks_completed, 1)) * 0.25
    weekend_factor = (data.weekend_work / 4) * 0.15
    
    burnout_score = min(
        workload_factor + hours_factor + deadline_factor + weekend_factor,
        1.0
    )
    
    # Determine risk level
    if burnout_score < 0.4:
        risk_level = "low"
    elif burnout_score < 0.7:
        risk_level = "medium"
    else:
        risk_level = "high"
    
    recommendations = []
    if data.avg_working_hours > 9:
        recommendations.append("Reduce working hours")
    if data.tasks_pending > data.tasks_completed:
        recommendations.append("Redistribute workload")
    if data.missed_deadlines > 3:
        recommendations.append("Review task priorities and deadlines")
    if data.weekend_work > 2:
        recommendations.append("Minimize weekend work")
    
    return {
        "user_id": data.user_id,
        "burnout_risk": risk_level,
        "score": round(burnout_score, 2),
        "recommendations": recommendations
    }

@app.post("/classify/ticket")
async def classify_ticket(data: TicketClassificationInput):
    """Classify support ticket and determine routing"""
    
    text = f"{data.title} {data.description}".lower()
    
    # Simple keyword-based classification
    if any(word in text for word in ["login", "password", "access", "403", "401"]):
        category = "access_control"
        department = "IT"
        priority = "high"
    elif any(word in text for word in ["bug", "error", "crash", "500"]):
        category = "technical"
        department = "Engineering"
        priority = "high"
    elif any(word in text for word in ["feature", "enhancement", "request"]):
        category = "feature_request"
        department = "Product"
        priority = "medium"
    elif any(word in text for word in ["report", "dashboard", "analytics"]):
        category = "reporting"
        department = "Analytics"
        priority = "medium"
    else:
        category = "general"
        department = "Support"
        priority = "low"
    
    return {
        "category": category,
        "priority": priority,
        "department": department,
        "confidence": 0.85
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Configuration

### Environment Variables

**Backend (.env):**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_secure_jwt_secret_key_here
JWT_EXPIRES_IN=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env):**
```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:3000,http://localhost:5000
```

**Frontend (.env):**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_ENABLE_ANALYTICS=true
```

## Common Patterns

### Protected API Calls

```javascript
// utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

api.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Real-time Notifications

```javascript
// hooks/useNotifications.js
import { useEffect, useState } from 'react';
import io from 'socket.io-client';

export const useNotifications = () => {
  const [notifications, setNotifications] = useState([]);
  const [socket, setSocket] = useState(null);

  useEffect(() => {
    const newSocket = io(process.env.REACT_APP_API_URL, {
      auth: { token: localStorage.getItem('token') }
    });

    newSocket.on('notification', (data) => {
      setNotifications(prev => [data, ...prev]);
    });

    setSocket(newSocket);

    return () => newSocket.close();
  }, []);

  return { notifications, socket };
};
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Verify connection string
echo $MONGODB_URI
```

### JWT Token Errors

```javascript
// Verify token in backend
const jwt = require('jsonwebtoken');

try {
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  console.log('Token valid:', decoded);
} catch (error) {
  console.error('Token invalid:', error.message);
}
```

### ML Service Not Responding

```bash
# Check service is running
curl http://localhost:8000/health

# View logs
tail -f ml-service/logs/service.log

# Restart service
cd ml-service
uvicorn main:app --reload --log-level debug
```

### CORS Issues

```javascript
// Backend: Configure CORS properly
const cors = require('cors');

app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true
}));
```

### Performance Issues

```javascript
// Add database indexes
// In MongoDB shell or migration:
db.users.createIndex({ email: 1 }, { unique: true });
db.tasks.createIndex
