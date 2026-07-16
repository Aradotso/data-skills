---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, task management, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "integrate AI-powered user management system"
  - "implement task tracking with burnout detection"
  - "create admin dashboard with user analytics"
  - "build user management system with ticket routing"
  - "add AI risk detection to user management"
  - "deploy enterprise user management application"
  - "configure JWT authentication for user system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that provides centralized user, task, and ticket management with integrated AI features including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing. Built with React frontend, Node.js backend, MongoDB database, and FastAPI ML service.

## Installation

### Prerequisites

Ensure you have Node.js (v14+), Python (3.8+), and MongoDB installed.

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-management
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
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
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

Application runs at `http://localhost:3000`

## Architecture

### Three-Tier Structure

1. **Frontend** (React) - User interface and admin dashboards
2. **Backend** (Node.js/Express) - REST API, authentication, business logic
3. **ML Service** (FastAPI) - AI analytics and predictions

## Key API Endpoints

### Authentication

```javascript
// Register user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@company.com",
  "password": "securePassword123",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securePassword123"
}
// Returns: { token, user }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer <JWT_TOKEN>" }

// Create user
POST /api/users
{
  "name": "Jane Smith",
  "email": "jane@company.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
{
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:userId
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks/user/:userId

// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Add new authentication flow",
  "assignedTo": "userId",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:taskId/status
{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:taskId/time
{
  "timeSpent": 3600 // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// Get tickets (admin)
GET /api/tickets

// Update ticket
PATCH /api/tickets/:ticketId
{
  "status": "resolved",
  "assignedTo": "adminId"
}
```

### AI Analytics Endpoints

```javascript
// Risk prediction
POST /api/ai/risk-prediction
{
  "userId": "userId",
  "activityData": {
    "loginAttempts": 5,
    "failedLogins": 3,
    "taskCompletionRate": 0.45,
    "averageResponseTime": 7200
  }
}

// Burnout detection
POST /api/ai/burnout-detection
{
  "userId": "userId",
  "workloadMetrics": {
    "tasksAssigned": 15,
    "hoursWorked": 55,
    "overtimeHours": 10,
    "weekendWork": true
  }
}

// Ticket classification
POST /api/ai/classify-ticket
{
  "title": "Cannot login",
  "description": "Getting 401 error"
}
// Returns: { category: "technical", priority: "high", suggestedAssignee: "techSupportId" }

// Anomaly detection
POST /api/ai/detect-anomaly
{
  "userId": "userId",
  "behavior": {
    "loginTime": "03:00 AM",
    "dataAccessed": 5000,
    "unusualIPs": ["192.168.1.100"]
  }
}
```

## Frontend Integration

### Authentication Setup

```javascript
// src/utils/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const api = axios.create({
  baseURL: API_URL,
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle auth errors
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
```

### User Dashboard Component

```javascript
// src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import { api } from '../utils/api';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const userId = JSON.parse(localStorage.getItem('user')).id;
      const response = await api.get(`/api/tasks/user/${userId}`);
      
      // Group tasks by status
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await api.patch(`/api/tasks/${taskId}/status`, { status: newStatus });
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="kanban-board">
      <div className="column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => (
          <TaskCard 
            key={task._id} 
            task={task} 
            onStatusChange={updateTaskStatus}
          />
        ))}
      </div>
      
      <div className="column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => (
          <TaskCard 
            key={task._id} 
            task={task} 
            onStatusChange={updateTaskStatus}
          />
        ))}
      </div>
      
      <div className="column">
        <h3>Done ({tasks.done.length})</h3>
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
```

### Admin Analytics Component

```javascript
// src/components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { api } from '../utils/api';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    fetchAnalytics();
    fetchAIAlerts();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const response = await api.get('/api/analytics/overview');
      setAnalytics(response.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const fetchAIAlerts = async () => {
    try {
      const response = await api.get('/api/ai/alerts');
      setAlerts(response.data);
    } catch (error) {
      console.error('Error fetching alerts:', error);
    }
  };

  const checkUserRisk = async (userId) => {
    try {
      const response = await api.post('/api/ai/risk-prediction', { userId });
      alert(`Risk Score: ${response.data.riskScore}\nLevel: ${response.data.riskLevel}`);
    } catch (error) {
      console.error('Error checking risk:', error);
    }
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="analytics-dashboard">
      <h2>Organization Analytics</h2>
      
      <div className="metrics-grid">
        <div className="metric-card">
          <h3>Total Users</h3>
          <p className="metric-value">{analytics.totalUsers}</p>
        </div>
        
        <div className="metric-card">
          <h3>Active Tasks</h3>
          <p className="metric-value">{analytics.activeTasks}</p>
        </div>
        
        <div className="metric-card">
          <h3>Open Tickets</h3>
          <p className="metric-value">{analytics.openTickets}</p>
        </div>
        
        <div className="metric-card">
          <h3>Completion Rate</h3>
          <p className="metric-value">{analytics.completionRate}%</p>
        </div>
      </div>

      <div className="ai-alerts">
        <h3>AI Alerts</h3>
        {alerts.map(alert => (
          <div key={alert.id} className={`alert alert-${alert.severity}`}>
            <strong>{alert.type}</strong>: {alert.message}
            {alert.userId && (
              <button onClick={() => checkUserRisk(alert.userId)}>
                Check Risk
              </button>
            )}
          </div>
        ))}
      </div>
    </div>
  );
};
```

## Backend Implementation

### Express Server Setup

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const authRoutes = require('./routes/auth');
const userRoutes = require('./routes/users');
const taskRoutes = require('./routes/tasks');
const ticketRoutes = require('./routes/tickets');
const aiRoutes = require('./routes/ai');

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Database connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
})
.then(() => console.log('MongoDB connected'))
.catch(err => console.error('MongoDB connection error:', err));

// Routes
app.use('/api/auth', authRoutes);
app.use('/api/users', userRoutes);
app.use('/api/tasks', taskRoutes);
app.use('/api/tickets', ticketRoutes);
app.use('/api/ai', aiRoutes);

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

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
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
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
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user || user.status !== 'active') {
      return res.status(401).json({ error: 'Invalid authentication' });
    }

    req.user = user;
    req.userId = user._id;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

exports.isAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};
```

### Task Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authenticate, isAdmin } = require('../middleware/auth');

// Get user's tasks
router.get('/user/:userId', authenticate, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task (admin only)
router.post('/', authenticate, isAdmin, async (req, res) => {
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
router.patch('/:taskId/status', authenticate, async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.taskId,
      { status: req.body.status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Track time
router.post('/:taskId/time', authenticate, async (req, res) => {
  try {
    const task = await Task.findById(req.params.taskId);
    task.timeSpent += req.body.timeSpent;
    task.updatedAt = Date.now();
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, Dict, List
import numpy as np
from sklearn.ensemble import RandomForestClassifier
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
MODEL_PATH = os.getenv("MODEL_PATH", "./models")
os.makedirs(MODEL_PATH, exist_ok=True)

class RiskPredictionRequest(BaseModel):
    userId: str
    activityData: Dict

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workloadMetrics: Dict

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class AnomalyDetectionRequest(BaseModel):
    userId: str
    behavior: Dict

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on activity patterns"""
    try:
        activity = request.activityData
        
        # Extract features
        features = [
            activity.get('loginAttempts', 0),
            activity.get('failedLogins', 0),
            activity.get('taskCompletionRate', 1.0),
            activity.get('averageResponseTime', 0) / 3600,  # hours
        ]
        
        # Simple risk calculation (can be replaced with ML model)
        risk_score = 0
        
        if activity.get('failedLogins', 0) > 3:
            risk_score += 30
        
        completion_rate = activity.get('taskCompletionRate', 1.0)
        if completion_rate < 0.5:
            risk_score += 25
        
        avg_response = activity.get('averageResponseTime', 0)
        if avg_response > 7200:  # > 2 hours
            risk_score += 20
        
        risk_level = 'low'
        if risk_score > 60:
            risk_level = 'high'
        elif risk_score > 30:
            risk_level = 'medium'
        
        return {
            "userId": request.userId,
            "riskScore": min(risk_score, 100),
            "riskLevel": risk_level,
            "factors": {
                "failedLogins": activity.get('failedLogins', 0),
                "completionRate": completion_rate,
                "responseTime": avg_response
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    """Detect employee burnout risk"""
    try:
        metrics = request.workloadMetrics
        
        burnout_score = 0
        
        # Tasks assigned
        tasks = metrics.get('tasksAssigned', 0)
        if tasks > 12:
            burnout_score += 25
        
        # Hours worked
        hours = metrics.get('hoursWorked', 40)
        if hours > 50:
            burnout_score += 30
        
        # Overtime
        overtime = metrics.get('overtimeHours', 0)
        if overtime > 5:
            burnout_score += 20
        
        # Weekend work
        if metrics.get('weekendWork', False):
            burnout_score += 25
        
        burnout_level = 'low'
        if burnout_score > 70:
            burnout_level = 'high'
        elif burnout_score > 40:
            burnout_level = 'medium'
        
        recommendations = []
        if burnout_level in ['medium', 'high']:
            recommendations.append("Redistribute workload")
            recommendations.append("Schedule time off")
            recommendations.append("Reduce overtime hours")
        
        return {
            "userId": request.userId,
            "burnoutScore": burnout_score,
            "burnoutLevel": burnout_level,
            "recommendations": recommendations,
            "metrics": metrics
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and suggest routing"""
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple keyword-based classification
        category = "general"
        priority = "medium"
        
        # Technical issues
        if any(word in text for word in ['login', 'password', 'access', 'error', '401', '500']):
            category = "technical"
            priority = "high"
        
        # Account issues
        elif any(word in text for word in ['account', 'profile', 'settings']):
            category = "account"
            priority = "medium"
        
        # Feature requests
        elif any(word in text for word in ['feature', 'request', 'add', 'improve']):
            category = "feature-request"
            priority = "low"
        
        # Bug reports
        elif any(word in text for word in ['bug', 'broken', 'not working', 'issue']):
            category = "bug"
            priority = "high"
        
        # Suggest assignee based on category
        suggested_assignee = {
            "technical": "tech-support",
            "account": "account-manager",
            "feature-request": "product-manager",
            "bug": "engineering",
            "general": "customer-support"
        }.get(category, "customer-support")
        
        return {
            "category": category,
            "priority": priority,
            "suggestedAssignee": suggested_assignee,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    try:
        behavior = request.behavior
        
        anomalies = []
        anomaly_score = 0
        
        # Unusual login time
        login_time = behavior.get('loginTime', '')
        if any(hour in login_time for hour in ['01:', '02:', '03:', '04:', '05:']):
            anomalies.append("Unusual login time detected")
            anomaly_score += 30
        
        # Excessive data access
        data_accessed = behavior.get('dataAccessed', 0)
        if data_accessed > 1000:
            anomalies.append("Excessive data access")
            anomaly_score += 40
        
        # Unusual IPs
        unusual_ips = behavior.get('unusualIPs', [])
        if unusual_ips:
            anomalies.append(f"Login from unusual IP(s): {', '.join(unusual_ips)}")
            anomaly_score += 30
        
        is_anomalous = anomaly_score > 50
        
        return {
            "userId": request.userId,
            "isAnomalous": is_anomalous,
            "anomalyScore": anomaly_score,
            "anomalies": anomalies,
            "recommendation": "Review user activity" if is_anomalous else "No action needed"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Configuration

### Environment Variables

**Backend (.env)**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-management
JWT_SECRET=your_secure_jwt_secret_key_here
JWT_EXPIRES_IN=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=production
CORS_ORIGIN=http://localhost:3000
```

**Frontend (.env)**
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENV=development
```

**ML Service (.env)**
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
PYTHONUNBUFFERED=1
```

### MongoDB Indexes

```javascript
// Run in MongoDB shell
use enterprise-user-management;

// User indexes
db.users.createIndex({ email: 1 }, { unique: true });
db.users.createIndex({ status: 1 });

// Task indexes
db.tasks.createIndex({ assignedTo: 1, status: 1 });
db.tasks.createIndex({ dueDate: 1 });

// Ticket indexes
db.tickets.createIndex({ userId: 1, status: 1 });
db.tickets.createIndex({ createdAt: -1 });
```

## Common Patterns

### Protected Route Component

```javascript
// src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Time Tracking Hook

```javascript
// src/hooks/useTimeTracker.js
import { useState, useEffect, useRef } from 'react';
import { api } from '../utils/api';

export const useTimeTracker = (taskId) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else {
      clearInterval(intervalRef.current);
    }

    return () => clearInterval(intervalRef.current);
  }, [isRunning]);

  const start = () => setIsRunning(true);
  const pause = () => setIsRunning(false);
  
  const stop = async () => {
    setIsRunning(false);
    if (seconds > 0) {
      try {
        await api.post(`/api/tasks/${taskId}/time`, { timeSpent: seconds });
        setSeconds(0);
      } catch (error) {
        console.error('Error saving time:', error);
      }
    }
  };

  const formatTime = () => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return { seconds, isRunning, start, pause, stop, formatTime };
};
```

### Real-time Notifications

```javascript
// src/utils/notifications.js
export class NotificationService {
  constructor() {
    this.listeners = [];
  }

  subscribe(callback) {
    this.listeners.push(callback);
    return () => {
      this.listeners = this.listeners.filter(cb => cb !== callback);
    };
  }

  notify(notification) {
    this.listeners.forEach(callback => callback(notification));
  }

  async checkAlerts() {
    try {
      const response = await api.get('/api/ai/alerts');
      response.data.forEach(alert => {
        this.notify({
          type: alert.severity,
          title: alert.type,
          message: alert.message
        });
      });
    } catch (error) {
      console.error('Error checking alerts:', error);
    }
  }
}

export const notificationService = new NotificationService();
```

## Troubleshooting

### MongoDB Connection Issues
