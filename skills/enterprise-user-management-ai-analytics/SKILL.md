---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "build user management dashboard with AI"
  - "implement task tracking with burnout detection"
  - "create admin panel with anomaly detection"
  - "add AI-powered ticket classification"
  - "deploy user management system with ML"
  - "configure enterprise task management app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with AI-powered insights. It provides role-based access control, Kanban-style task tracking, support ticket management, and ML-driven analytics including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Key capabilities:**
- JWT-based authentication with role-based access control (Admin/User)
- Task management with Kanban board and time tracking
- Support ticket system with AI classification and routing
- AI analytics: risk detection, anomaly detection, burnout prediction, project delay forecasting
- Real-time dashboards for users and administrators
- MongoDB data persistence

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+ and pip
- MongoDB (local or Atlas)

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup (Node.js)

```bash
cd backend
npm install
```

Create `.env` file:

```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Runs on http://localhost:5000
```

### ML Service Setup (Python/FastAPI)

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# Runs on http://localhost:8000
```

### Frontend Setup (React)

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

## Key API Endpoints

### Authentication

**Login:**
```javascript
// POST /api/auth/login
const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'admin@example.com',
    password: 'password123'
  })
});

const { token, user } = await response.json();
// Store token for authenticated requests
localStorage.setItem('authToken', token);
```

**Register User:**
```javascript
// POST /api/auth/register
const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/register`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name: 'John Doe',
    email: 'john@example.com',
    password: 'securepass',
    role: 'user' // 'admin' or 'user'
  })
});
```

### User Management (Admin)

**Get All Users:**
```javascript
// GET /api/users
const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`
  }
});

const users = await response.json();
```

**Update User:**
```javascript
// PUT /api/users/:userId
await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'Jane Doe',
    role: 'admin',
    status: 'active'
  })
});
```

**Delete User:**
```javascript
// DELETE /api/users/:userId
await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
  method: 'DELETE',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`
  }
});
```

### Task Management

**Create Task:**
```javascript
// POST /api/tasks
const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: 'Implement user authentication',
    description: 'Add JWT-based authentication',
    assignedTo: 'userId123',
    status: 'todo', // 'todo', 'in-progress', 'done'
    priority: 'high', // 'low', 'medium', 'high'
    dueDate: '2026-05-01'
  })
});
```

**Update Task Status:**
```javascript
// PUT /api/tasks/:taskId
await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    status: 'in-progress',
    timeSpent: 120 // minutes
  })
});
```

**Get User Tasks:**
```javascript
// GET /api/tasks/user/:userId
const response = await fetch(
  `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
  {
    headers: {
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    }
  }
);

const tasks = await response.json();
```

### Support Tickets

**Create Ticket:**
```javascript
// POST /api/tickets
const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('authToken')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    subject: 'Cannot access dashboard',
    description: 'Getting 403 error when accessing admin dashboard',
    priority: 'high',
    category: 'technical' // AI will auto-classify if not provided
  })
});
```

**AI Ticket Classification:**
```javascript
// POST /api/ml/classify-ticket
const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/classify-ticket`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    subject: 'Cannot login to my account',
    description: 'Forgot password and reset link not working'
  })
});

const { category, priority, suggestedAssignee } = await response.json();
// category: 'authentication', 'technical', 'billing', etc.
```

### AI Analytics

**Risk Prediction:**
```javascript
// POST /api/ml/risk-prediction
const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/risk-prediction`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: 'user123',
    taskCompletionRate: 0.65,
    averageTaskDelay: 2.5, // days
    ticketCount: 8,
    loginFrequency: 15 // per week
  })
});

const { riskScore, riskLevel, factors } = await response.json();
// riskLevel: 'low', 'medium', 'high'
```

**Anomaly Detection:**
```javascript
// POST /api/ml/anomaly-detection
const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/anomaly-detection`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: 'user123',
    loginTime: '2026-04-15T03:30:00Z',
    ipAddress: '192.168.1.100',
    location: 'New York',
    deviceType: 'mobile'
  })
});

const { isAnomaly, anomalyScore, reason } = await response.json();
```

**Burnout Detection:**
```javascript
// POST /api/ml/burnout-detection
const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: 'user123',
    weeklyWorkHours: 55,
    taskCount: 25,
    overdueTasksCount: 8,
    ticketResponseTime: 180, // minutes
    lastBreakDays: 45
  })
});

const { burnoutRisk, score, recommendations } = await response.json();
// burnoutRisk: 'low', 'medium', 'high'
```

**Project Delay Prediction:**
```javascript
// POST /api/ml/project-delay-prediction
const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/project-delay-prediction`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    projectId: 'proj123',
    totalTasks: 50,
    completedTasks: 20,
    overdueTasks: 5,
    teamSize: 6,
    daysRemaining: 30
  })
});

const { delayProbability, estimatedDelay, riskFactors } = await response.json();
```

## Common Patterns

### Protected Route Component (React)

```javascript
// components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('authToken');
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

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
      {
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('authToken')}`
        }
      }
    );
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
      method: 'PUT',
      headers: {
        'Authorization': `Bearer ${localStorage.getItem('authToken')}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} onMove={moveTask} targetStatus="in-progress" />
      <Column title="In Progress" tasks={tasks.inProgress} onMove={moveTask} targetStatus="done" />
      <Column title="Done" tasks={tasks.done} />
    </div>
  );
};
```

### Backend API Route (Express.js)

```javascript
// routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authenticateToken, isAdmin } = require('../middleware/auth');

// Get all tasks (admin only)
router.get('/', authenticateToken, isAdmin, async (req, res) => {
  try {
    const tasks = await Task.find().populate('assignedTo', 'name email');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user's tasks
router.get('/user/:userId', authenticateToken, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authenticateToken, isAdmin, async (req, res) => {
  try {
    const task = new Task(req.body);
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update task
router.put('/:taskId', authenticateToken, async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.taskId,
      req.body,
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### ML Service Endpoint (FastAPI)

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from river import anomaly, compose, preprocessing

app = FastAPI()

# Load or initialize models
try:
    risk_model = joblib.load('models/risk_model.pkl')
except:
    risk_model = None

# Anomaly detector using River (online learning)
anomaly_detector = compose.Pipeline(
    preprocessing.StandardScaler(),
    anomaly.HalfSpaceTrees()
)

class RiskPredictionRequest(BaseModel):
    userId: str
    taskCompletionRate: float
    averageTaskDelay: float
    ticketCount: int
    loginFrequency: int

class AnomalyDetectionRequest(BaseModel):
    userId: str
    loginTime: str
    ipAddress: str
    location: str
    deviceType: str

@app.post("/api/ml/risk-prediction")
async def predict_risk(request: RiskPredictionRequest):
    try:
        features = np.array([[
            request.taskCompletionRate,
            request.averageTaskDelay,
            request.ticketCount,
            request.loginFrequency
        ]])
        
        if risk_model:
            risk_score = risk_model.predict_proba(features)[0][1]
        else:
            # Fallback heuristic
            risk_score = (
                (1 - request.taskCompletionRate) * 0.4 +
                (request.averageTaskDelay / 10) * 0.3 +
                (request.ticketCount / 20) * 0.3
            )
        
        risk_level = 'high' if risk_score > 0.7 else 'medium' if risk_score > 0.4 else 'low'
        
        return {
            "riskScore": float(risk_score),
            "riskLevel": risk_level,
            "factors": {
                "taskCompletion": request.taskCompletionRate,
                "delays": request.averageTaskDelay,
                "tickets": request.ticketCount
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        # Feature engineering
        hour = int(request.loginTime.split('T')[1].split(':')[0])
        is_night = 1 if hour < 6 or hour > 22 else 0
        is_mobile = 1 if request.deviceType == 'mobile' else 0
        
        features = {
            'hour': hour,
            'is_night': is_night,
            'is_mobile': is_mobile
        }
        
        # Online anomaly detection
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.6
        
        return {
            "isAnomaly": is_anomaly,
            "anomalyScore": float(score),
            "reason": "Unusual login time" if is_night else "Normal activity"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: dict):
    try:
        weekly_hours = request['weeklyWorkHours']
        task_count = request['taskCount']
        overdue_count = request['overdueTasksCount']
        last_break = request['lastBreakDays']
        
        # Burnout score calculation
        score = (
            (weekly_hours / 60) * 0.3 +
            (task_count / 30) * 0.2 +
            (overdue_count / 10) * 0.3 +
            (last_break / 60) * 0.2
        )
        
        burnout_risk = 'high' if score > 0.7 else 'medium' if score > 0.4 else 'low'
        
        recommendations = []
        if weekly_hours > 50:
            recommendations.append("Reduce weekly work hours")
        if last_break > 30:
            recommendations.append("Take a break soon")
        if overdue_count > 5:
            recommendations.append("Prioritize overdue tasks")
        
        return {
            "burnoutRisk": burnout_risk,
            "score": float(score),
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### MongoDB Schema Models

```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date
});

module.exports = mongoose.model('User', userSchema);
```

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
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
  timeSpent: { type: Number, default: 0 }, // minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  category: String, // AI-classified
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Configuration

### Environment Variables

**Backend (.env):**
```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_secure_jwt_secret
JWT_EXPIRATION=24h
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env):**
```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
ENABLE_MODEL_TRAINING=true
MODEL_UPDATE_INTERVAL=3600
```

**Frontend (.env):**
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_AI_FEATURES=true
```

## Troubleshooting

### Authentication Issues

**JWT Token Expired:**
```javascript
// Add token refresh logic
const refreshToken = async () => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/refresh`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    }
  });
  
  if (response.ok) {
    const { token } = await response.json();
    localStorage.setItem('authToken', token);
    return token;
  }
  
  // Logout if refresh fails
  localStorage.removeItem('authToken');
  window.location.href = '/login';
};
```

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
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

### ML Model Not Loading

```python
# ml-service/models/loader.py
import os
import joblib
from pathlib import Path

def load_or_create_model(model_name):
    model_path = Path(os.getenv('MODEL_PATH', './models')) / f'{model_name}.pkl'
    
    if model_path.exists():
        return joblib.load(model_path)
    else:
        print(f"Model {model_name} not found, using fallback heuristics")
        return None
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### Rate Limiting for API Protection

```javascript
// backend/middleware/rateLimit.js
const rateLimit = require('express-rate-limit');

const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  message: 'Too many requests from this IP'
});

app.use('/api/', apiLimiter);
```
