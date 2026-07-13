---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, and burnout prediction
triggers:
  - how do I set up the enterprise user management system
  - implement AI analytics for user behavior monitoring
  - integrate risk detection and anomaly detection features
  - build a user management dashboard with AI insights
  - add burnout prediction to user management
  - create ticket classification system with ML
  - set up Kanban board with time tracking
  - implement JWT authentication for user management
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack application that combines traditional user management with AI-powered analytics. It provides user authentication, task management with Kanban boards, support ticket systems, and AI features including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

**Architecture:**
- **Frontend**: React.js application
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based microservice using scikit-learn and River
- **Database**: MongoDB

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB installed and running
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
LOG_LEVEL=INFO
EOF

uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

npm start
```

## Key API Endpoints

### Authentication

```javascript
// Login
POST /api/auth/login
Body: { email, password }
Response: { token, user }

// Register
POST /api/auth/register
Body: { name, email, password, role }
Response: { token, user }

// Verify Token
GET /api/auth/verify
Headers: { Authorization: "Bearer <token>" }
Response: { user }
```

### User Management

```javascript
// Get all users (Admin)
GET /api/users
Headers: { Authorization: "Bearer <token>" }
Response: { users: [...] }

// Get user by ID
GET /api/users/:id
Headers: { Authorization: "Bearer <token>" }
Response: { user }

// Update user
PUT /api/users/:id
Headers: { Authorization: "Bearer <token>" }
Body: { name, email, role, status }
Response: { user }

// Delete user
DELETE /api/users/:id
Headers: { Authorization: "Bearer <token>" }
Response: { message }
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { Authorization: "Bearer <token>" }
Response: { tasks: [...] }

// Create task
POST /api/tasks
Headers: { Authorization: "Bearer <token>" }
Body: { title, description, assignedTo, status, priority, deadline }
Response: { task }

// Update task status
PUT /api/tasks/:id
Headers: { Authorization: "Bearer <token>" }
Body: { status, timeSpent }
Response: { task }
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { Authorization: "Bearer <token>" }
Body: { subject, description, priority }
Response: { ticket }

// Get tickets
GET /api/tickets
Headers: { Authorization: "Bearer <token>" }
Query: { status, priority }
Response: { tickets: [...] }

// AI Classification
POST /api/tickets/:id/classify
Headers: { Authorization: "Bearer <token>" }
Response: { category, priority, assignedDepartment }
```

### AI Analytics

```javascript
// Risk Prediction
POST /api/ml/risk-prediction
Headers: { Authorization: "Bearer <token>" }
Body: { userId, behaviorMetrics }
Response: { riskScore, factors, recommendations }

// Anomaly Detection
POST /api/ml/anomaly-detection
Headers: { Authorization: "Bearer <token>" }
Body: { userId, activityLogs }
Response: { isAnomaly, confidence, anomalyType }

// Burnout Detection
POST /api/ml/burnout-detection
Headers: { Authorization: "Bearer <token>" }
Body: { userId, workloadData }
Response: { burnoutRisk, factors, suggestions }

// Project Insights
POST /api/ml/project-insights
Headers: { Authorization: "Bearer <token>" }
Body: { projectId, metrics }
Response: { delayPrediction, completionEstimate, risks }
```

## Frontend Implementation

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      verifyToken(token);
    } else {
      setLoading(false);
    }
  }, []);

  const verifyToken = async (token) => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/verify`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setUser(res.data.user);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', res.data.token);
    setUser(res.data.user);
    return res.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [timer, setTimer] = useState({ taskId: null, seconds: 0, isRunning: false });

  useEffect(() => {
    fetchTasks();
  }, []);

  useEffect(() => {
    let interval;
    if (timer.isRunning) {
      interval = setInterval(() => {
        setTimer(prev => ({ ...prev, seconds: prev.seconds + 1 }));
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [timer.isRunning]);

  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      const grouped = {
        todo: res.data.tasks.filter(t => t.status === 'todo'),
        inProgress: res.data.tasks.filter(t => t.status === 'in-progress'),
        done: res.data.tasks.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus, timeSpent = 0) => {
    try {
      const token = localStorage.getItem('token');
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`,
        { status: newStatus, timeSpent },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const startTimer = (taskId) => {
    setTimer({ taskId, seconds: 0, isRunning: true });
  };

  const stopTimer = () => {
    if (timer.taskId) {
      updateTaskStatus(timer.taskId, 'in-progress', timer.seconds);
    }
    setTimer({ taskId: null, seconds: 0, isRunning: false });
  };

  const formatTime = (seconds) => {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    const s = seconds % 60;
    return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status === 'inProgress' ? 'In Progress' : status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>{task.priority}</span>
              {timer.taskId === task._id ? (
                <div className="timer">
                  <span>{formatTime(timer.seconds)}</span>
                  <button onClick={stopTimer}>Stop</button>
                </div>
              ) : (
                status === 'todo' && (
                  <button onClick={() => startTimer(task._id)}>Start</button>
                )
              )}
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

## Backend Implementation

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      throw new Error();
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');

    if (!user) {
      throw new Error();
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Please authenticate' });
  }
};

const adminAuth = async (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { auth, adminAuth };
```

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const bcrypt = require('bcryptjs');

exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ users });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.getUserById = async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json({ user });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.updateUser = async (req, res) => {
  try {
    const { name, email, role, status } = req.body;
    const updateData = { name, email, role, status };

    const user = await User.findByIdAndUpdate(
      req.params.id,
      updateData,
      { new: true, runValidators: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json({ user });
  } catch (error) {
    res.status(400).json({ message: 'Update failed', error: error.message });
  }
};

exports.deleteUser = async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    required: true
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
  deadline: {
    type: Date
  },
  timeSpent: {
    type: Number,
    default: 0
  },
  tags: [String]
}, {
  timestamps: true
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
from typing import List, Dict, Optional
import numpy as np
from sklearn.ensemble import IsolationForest, RandomForestClassifier
from river import anomaly, ensemble
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

# Models
anomaly_detector = anomaly.HalfSpaceTrees(seed=42)
risk_model = None
burnout_model = None

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

class RiskPredictionRequest(BaseModel):
    userId: str
    behaviorMetrics: Dict[str, float]

class AnomalyDetectionRequest(BaseModel):
    userId: str
    activityLogs: List[Dict[str, any]]

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workloadData: Dict[str, float]

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str
    metadata: Optional[Dict] = {}

@app.post("/risk-prediction")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavior metrics"""
    try:
        metrics = request.behaviorMetrics
        
        # Extract features
        features = np.array([
            metrics.get('loginFrequency', 0),
            metrics.get('failedLoginAttempts', 0),
            metrics.get('dataAccessVolume', 0),
            metrics.get('unusualAccessTime', 0),
            metrics.get('permissionChanges', 0)
        ]).reshape(1, -1)
        
        # Simple rule-based risk scoring
        risk_score = 0.0
        factors = []
        
        if metrics.get('failedLoginAttempts', 0) > 5:
            risk_score += 0.3
            factors.append("High failed login attempts")
        
        if metrics.get('unusualAccessTime', 0) > 0.7:
            risk_score += 0.25
            factors.append("Unusual access time patterns")
        
        if metrics.get('permissionChanges', 0) > 3:
            risk_score += 0.2
            factors.append("Multiple permission changes")
        
        if metrics.get('dataAccessVolume', 0) > 1000:
            risk_score += 0.25
            factors.append("High data access volume")
        
        risk_level = "low" if risk_score < 0.3 else "medium" if risk_score < 0.6 else "high"
        
        recommendations = []
        if risk_score > 0.5:
            recommendations.append("Enable two-factor authentication")
            recommendations.append("Review recent access logs")
            recommendations.append("Consider account verification")
        
        return {
            "riskScore": min(risk_score, 1.0),
            "riskLevel": risk_level,
            "factors": factors,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/anomaly-detection")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalies in user activity"""
    try:
        activities = request.activityLogs
        
        if not activities:
            return {"isAnomaly": False, "confidence": 0.0, "anomalyType": None}
        
        # Extract features from activity logs
        features = []
        for activity in activities:
            feature_vector = {
                'timestamp': activity.get('timestamp', 0),
                'action_type': hash(activity.get('action', '')) % 100,
                'resource_accessed': hash(activity.get('resource', '')) % 100,
                'duration': activity.get('duration', 0)
            }
            features.append(feature_vector)
        
        # Use online learning with River
        anomaly_scores = []
        for feat in features:
            score = anomaly_detector.score_one(feat)
            anomaly_detector.learn_one(feat)
            anomaly_scores.append(score)
        
        avg_score = np.mean(anomaly_scores)
        is_anomaly = avg_score > 0.7
        
        anomaly_type = None
        if is_anomaly:
            if any(a.get('action') == 'data_download' for a in activities):
                anomaly_type = "unusual_data_access"
            elif any(a.get('duration', 0) > 3600 for a in activities):
                anomaly_type = "extended_session"
            else:
                anomaly_type = "suspicious_behavior"
        
        return {
            "isAnomaly": bool(is_anomaly),
            "confidence": float(avg_score),
            "anomalyType": anomaly_type,
            "details": f"Analyzed {len(activities)} activities"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/burnout-detection")
async def detect_burnout(request: BurnoutDetectionRequest):
    """Detect employee burnout risk"""
    try:
        workload = request.workloadData
        
        # Calculate burnout indicators
        weekly_hours = workload.get('weeklyHours', 40)
        overtime_hours = workload.get('overtimeHours', 0)
        tasks_completed = workload.get('tasksCompleted', 0)
        tasks_pending = workload.get('tasksPending', 0)
        missed_deadlines = workload.get('missedDeadlines', 0)
        
        burnout_score = 0.0
        factors = []
        
        if weekly_hours > 50:
            burnout_score += 0.25
            factors.append(f"Working {weekly_hours} hours/week")
        
        if overtime_hours > 10:
            burnout_score += 0.2
            factors.append(f"{overtime_hours} hours of overtime")
        
        if tasks_pending > tasks_completed * 1.5:
            burnout_score += 0.2
            factors.append("High pending task ratio")
        
        if missed_deadlines > 2:
            burnout_score += 0.2
            factors.append(f"{missed_deadlines} missed deadlines")
        
        if workload.get('workLifeBalance', 5) < 3:
            burnout_score += 0.15
            factors.append("Low work-life balance score")
        
        risk_level = "low" if burnout_score < 0.3 else "medium" if burnout_score < 0.6 else "high"
        
        suggestions = []
        if burnout_score > 0.5:
            suggestions.append("Reduce overtime hours")
            suggestions.append("Redistribute pending tasks")
            suggestions.append("Schedule time off")
            suggestions.append("Consider workload rebalancing")
        
        return {
            "burnoutRisk": min(burnout_score, 1.0),
            "riskLevel": risk_level,
            "factors": factors,
            "suggestions": suggestions,
            "metrics": {
                "weeklyHours": weekly_hours,
                "taskRatio": tasks_pending / max(tasks_completed, 1)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ticket-classification")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and route to appropriate department"""
    try:
        text = f"{request.subject} {request.description}".lower()
        
        # Simple keyword-based classification
        categories = {
            "technical": ["error", "bug", "crash", "not working", "broken"],
            "access": ["password", "login", "access", "permission", "locked"],
            "billing": ["payment", "invoice", "billing", "subscription", "refund"],
            "feature": ["feature", "enhancement", "request", "add", "new"]
        }
        
        scores = {}
        for category, keywords in categories.items():
            score = sum(1 for keyword in keywords if keyword in text)
            scores[category] = score
        
        category = max(scores, key=scores.get) if max(scores.values()) > 0 else "general"
        
        # Determine priority based on keywords
        priority = "low"
        if any(word in text for word in ["urgent", "critical", "emergency", "down"]):
            priority = "high"
        elif any(word in text for word in ["important", "asap", "soon"]):
            priority = "medium"
        
        # Route to department
        department_map = {
            "technical": "IT Support",
            "access": "Security Team",
            "billing": "Finance",
            "feature": "Product Team",
            "general": "Customer Support"
        }
        
        return {
            "category": category,
            "priority": priority,
            "assignedDepartment": department_map.get(category, "Customer Support"),
            "confidence": scores.get(category, 0) / max(sum(scores.values()), 1)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

### Requirements File

```python
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
numpy==1.24.3
scikit-learn==1.3.2
river==0.19.0
joblib==1.3.2
python-dotenv==1.0.0
```

## Configuration

### MongoDB Connection

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Environment Variables

```bash
# Backend (.env)
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development

# Frontend (.env)
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000

# ML Service (.env)
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Common Patterns

### API Client Setup

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
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

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import api from '../utils/api';
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
      const [users, tasks, tickets] = await Promise.all([
        api.get('/api/users'),
        api.get('/api/tasks'),
        api.get('/api/tickets?status=open')
      ]);

      // Fetch AI risk predictions for users
      const riskPromises = users.data.users.slice(0, 10).map(user =>
        axios.post(`${process.env.REACT_APP_ML_API_URL}/risk-prediction`, {
          userId: user._id,
          behaviorMetrics: user.behaviorMetrics || {}
        }).catch(() => null)
      );

      const risks = await Promise.all(riskPromises);
      const highRisks = risks
        .filter(r => r && r.data.riskLevel === 'high')
        .map((r, i) => ({
          user: users.data.users[i].name,
          score: r.data.riskScore
        }));

      setAnalytics({
        totalUsers: users.data.users.length,
        activeTasks: tasks.data.tasks.filter(t => t.status !== 'done').length,
        openTickets: tickets.data.tickets.length,
        riskAlerts: highRisks
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h2>Admin Dashboard</h2>
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
        <div className="metric-card alert">
          <h3>Risk Alerts</h3>
          <p className="metric-value">{analytics.riskAlerts.length}</p>
        </div>
      </div>
      {analytics.riskAlerts.length > 0 && (
        <div className="risk-alerts">
          <h3>High Risk Users</h3>
          {analytics.riskAlerts.map((alert, i) => (
            <div key={i} className="alert-item">
              <span>{alert.user}</span>
              <span>Risk Score: {(alert.score * 100).toFixed(0)}%</span>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

export default AdminDashboard;
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// Check MongoDB is running
// Linux/Mac
sudo systemctl status mongod

// Start MongoDB
sudo systemctl start mongod

// Backend connection test
const mongoose = require('mongoose');
mongoose.connect(process.env.MONGODB_URI)
  .then(() => console.log('Connected'))
  .catch(err => console.error('Connection error:', err));
```

### JWT Token Expiration

```javascript
// Refresh token logic
const refreshToken = async () => {
  try {
    const res
