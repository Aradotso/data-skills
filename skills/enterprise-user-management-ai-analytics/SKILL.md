---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - how do I set up the enterprise user management system
  - help me integrate AI analytics into user management
  - show me how to implement task tracking with AI insights
  - configure user management with ML predictions
  - create an admin dashboard with AI features
  - implement burnout detection and risk analysis
  - build a kanban board with time tracking
  - set up JWT authentication for user management
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides role-based access control, Kanban task boards, support ticket management, and ML-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

**Key Components:**
- **Frontend**: React.js single-page application
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI microservice with scikit-learn and River
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites
```bash
# Node.js 14+, Python 3.8+, MongoDB 4.4+
node --version
python --version
mongod --version
```

### Full System Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env
# Configure MongoDB connection and JWT secret
npm start  # Runs on http://localhost:5000

# ML Service setup
cd ../ml-service
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup
cd ../frontend
npm install
npm start  # Runs on http://localhost:3000
```

## Configuration

### Backend Environment Variables (.env)

```bash
# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
DB_NAME=enterprise_user_mgmt

# Authentication
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRATION=24h
REFRESH_TOKEN_EXPIRATION=7d

# Server
PORT=5000
NODE_ENV=development

# ML Service
ML_SERVICE_URL=http://localhost:8000

# CORS
FRONTEND_URL=http://localhost:3000
```

### ML Service Configuration (ml-service/.env)

```bash
# Model paths
MODEL_PATH=./models
RISK_MODEL_PATH=./models/risk_model.pkl
ANOMALY_MODEL_PATH=./models/anomaly_model.pkl
BURNOUT_MODEL_PATH=./models/burnout_model.pkl

# API Settings
API_HOST=0.0.0.0
API_PORT=8000

# Logging
LOG_LEVEL=INFO
```

## Backend API Usage

### Authentication Endpoints

```javascript
// User Registration
const register = async (userData) => {
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

// User Login
const login = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin Only)

```javascript
// Get all users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// Update user role
const updateUserRole = async (userId, newRole) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ role: newRole })
  });
  return response.json();
};

// Delete user
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

```javascript
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
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// Update task status (Kanban board)
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
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
    body: JSON.stringify({ timeSpent: timeSpent }) // in minutes
  });
  return response.json();
};
```

### Support Ticket Management

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
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category // 'technical', 'billing', 'general'
    })
  });
  return response.json();
};

// Get ticket with AI classification
const getTicketWithAI = async (ticketId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}/analyze`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## ML Service API Usage

### Risk Prediction

```python
# FastAPI endpoint in ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np

app = FastAPI()

class UserBehavior(BaseModel):
    login_frequency: float
    failed_login_attempts: int
    unusual_hours_access: int
    data_access_volume: float
    permission_changes: int

@app.post("/api/ml/predict-risk")
async def predict_risk(behavior: UserBehavior):
    try:
        model = joblib.load('models/risk_model.pkl')
        features = np.array([[
            behavior.login_frequency,
            behavior.failed_login_attempts,
            behavior.unusual_hours_access,
            behavior.data_access_volume,
            behavior.permission_changes
        ]])
        risk_score = model.predict_proba(features)[0][1]
        return {
            "risk_score": float(risk_score),
            "risk_level": "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### JavaScript Client for Risk Prediction

```javascript
const predictUserRisk = async (userId) => {
  const token = localStorage.getItem('token');
  
  // Get user behavior data from backend
  const behaviorData = await fetch(`http://localhost:5000/api/users/${userId}/behavior`, {
    headers: { 'Authorization': `Bearer ${token}` }
  }).then(r => r.json());
  
  // Call ML service
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(behaviorData)
  });
  
  return response.json();
};
```

### Burnout Detection

```python
# ML Service endpoint
class WorkloadMetrics(BaseModel):
    tasks_completed: int
    avg_task_duration: float
    overtime_hours: float
    tasks_overdue: int
    days_without_break: int

@app.post("/api/ml/detect-burnout")
async def detect_burnout(metrics: WorkloadMetrics):
    try:
        model = joblib.load('models/burnout_model.pkl')
        features = np.array([[
            metrics.tasks_completed,
            metrics.avg_task_duration,
            metrics.overtime_hours,
            metrics.tasks_overdue,
            metrics.days_without_break
        ]])
        burnout_probability = model.predict_proba(features)[0][1]
        return {
            "burnout_score": float(burnout_probability),
            "status": "critical" if burnout_probability > 0.8 else "warning" if burnout_probability > 0.5 else "healthy",
            "recommendations": generate_recommendations(burnout_probability)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def generate_recommendations(score):
    if score > 0.8:
        return ["Immediate workload reduction needed", "Schedule mandatory break", "Manager intervention required"]
    elif score > 0.5:
        return ["Review task distribution", "Encourage time off", "Monitor progress"]
    return ["Maintain current pace", "Regular check-ins"]
```

### Anomaly Detection

```python
from river import anomaly, preprocessing

class AnomalyDetector:
    def __init__(self):
        self.model = preprocessing.StandardScaler() | anomaly.HalfSpaceTrees()
    
    def detect(self, features):
        score = self.model.score_one(features)
        self.model.learn_one(features)
        return score

detector = AnomalyDetector()

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(activity: dict):
    features = {
        'hour': activity['hour'],
        'login_count': activity['login_count'],
        'data_accessed': activity['data_accessed'],
        'api_calls': activity['api_calls']
    }
    score = detector.detect(features)
    return {
        "anomaly_score": float(score),
        "is_anomaly": score > 0.7,
        "severity": "high" if score > 0.9 else "medium" if score > 0.7 else "low"
    }
```

## Frontend React Components

### Protected Route with JWT

```javascript
// components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import './KanbanBoard.css';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  
  useEffect(() => {
    fetchTasks();
  }, [userId]);
  
  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`http://localhost:5000/api/tasks?userId=${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };
  
  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };
  
  const handleDrop = async (e, newStatus) => {
    const taskId = e.dataTransfer.getData('taskId');
    const token = localStorage.getItem('token');
    
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    
    fetchTasks();
  };
  
  const Column = ({ title, status, tasks }) => (
    <div 
      className="kanban-column"
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => handleDrop(e, status)}
    >
      <h3>{title}</h3>
      {tasks.map(task => (
        <div 
          key={task._id}
          className="task-card"
          draggable
          onDragStart={(e) => handleDragStart(e, task._id)}
        >
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority-${task.priority}`}>{task.priority}</span>
        </div>
      ))}
    </div>
  );
  
  return (
    <div className="kanban-board">
      <Column title="To Do" status="todo" tasks={tasks.todo} />
      <Column title="In Progress" status="in-progress" tasks={tasks.inProgress} />
      <Column title="Done" status="done" tasks={tasks.done} />
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';

const AIAnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    burnoutAlerts: [],
    anomalies: []
  });
  
  useEffect(() => {
    fetchAnalytics();
    const interval = setInterval(fetchAnalytics, 60000); // Refresh every minute
    return () => clearInterval(interval);
  }, []);
  
  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch('http://localhost:5000/api/analytics/ai-insights', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setAnalytics(data);
  };
  
  return (
    <div className="ai-dashboard">
      <section className="risk-analysis">
        <h2>High Risk Users</h2>
        {analytics.riskUsers.map(user => (
          <div key={user.id} className="alert-card">
            <h3>{user.username}</h3>
            <p>Risk Score: {(user.riskScore * 100).toFixed(1)}%</p>
            <span className="badge-danger">{user.riskLevel}</span>
          </div>
        ))}
      </section>
      
      <section className="burnout-alerts">
        <h2>Burnout Alerts</h2>
        {analytics.burnoutAlerts.map(alert => (
          <div key={alert.userId} className="alert-card">
            <h3>{alert.username}</h3>
            <p>Burnout Score: {(alert.score * 100).toFixed(1)}%</p>
            <ul>
              {alert.recommendations.map((rec, i) => (
                <li key={i}>{rec}</li>
              ))}
            </ul>
          </div>
        ))}
      </section>
      
      <section className="anomalies">
        <h2>Recent Anomalies</h2>
        {analytics.anomalies.map((anomaly, i) => (
          <div key={i} className="alert-card">
            <p>{anomaly.description}</p>
            <span className={`severity-${anomaly.severity}`}>{anomaly.severity}</span>
            <small>{new Date(anomaly.timestamp).toLocaleString()}</small>
          </div>
        ))}
      </section>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Common Patterns

### Setting up JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }
  
  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) {
      return res.status(403).json({ error: 'Invalid or expired token' });
    }
    req.user = user;
    next();
  });
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authenticateToken, requireAdmin };
```

### Real-time Notifications with WebSocket

```javascript
// backend/websocket.js
const WebSocket = require('ws');

const setupWebSocket = (server) => {
  const wss = new WebSocket.Server({ server });
  
  wss.on('connection', (ws, req) => {
    const userId = new URLSearchParams(req.url.split('?')[1]).get('userId');
    ws.userId = userId;
    
    ws.on('message', (message) => {
      // Handle incoming messages
    });
  });
  
  return wss;
};

const notifyUser = (wss, userId, notification) => {
  wss.clients.forEach(client => {
    if (client.userId === userId && client.readyState === WebSocket.OPEN) {
      client.send(JSON.stringify(notification));
    }
  });
};

module.exports = { setupWebSocket, notifyUser };
```

## Troubleshooting

### JWT Token Expiration
```javascript
// Implement token refresh
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await fetch('http://localhost:5000/api/auth/refresh', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};
```

### MongoDB Connection Issues
```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Model Loading Errors
```python
# ml-service/utils/model_loader.py
import joblib
import os
from pathlib import Path

def load_model(model_name):
    model_path = Path(os.getenv('MODEL_PATH', './models')) / f'{model_name}.pkl'
    if not model_path.exists():
        raise FileNotFoundError(f"Model {model_name} not found at {model_path}")
    return joblib.load(model_path)

def ensure_models_exist():
    required_models = ['risk_model', 'burnout_model', 'anomaly_model']
    for model in required_models:
        try:
            load_model(model)
        except FileNotFoundError:
            print(f"Warning: {model} not found. Training default model...")
            train_default_model(model)
```

### CORS Configuration
```javascript
// backend/app.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Performance Optimization for Large Datasets
```javascript
// Implement pagination
const getTasks = async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const skip = (page - 1) * limit;
  
  const tasks = await Task.find({ assignedTo: req.user.id })
    .skip(skip)
    .limit(limit)
    .sort({ createdAt: -1 });
  
  const total = await Task.countDocuments({ assignedTo: req.user.id });
  
  res.json({
    tasks,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit)
    }
  });
};
```
