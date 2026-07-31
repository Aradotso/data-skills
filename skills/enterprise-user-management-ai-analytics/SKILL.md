---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket classification, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with AI insights"
  - "build task tracking system with anomaly detection"
  - "integrate AI ticket classification system"
  - "develop user management with risk prediction"
  - "add burnout detection to user system"
  - "implement kanban board with AI analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack application for managing enterprise users, tasks, and support tickets with integrated AI analytics for risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

Enterprise User Management System provides:
- **User Management**: Role-based access control, authentication via JWT
- **Task Tracking**: Kanban boards, time tracking, task assignment
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Admin analytics and user performance insights

## Installation

### Prerequisites
- Node.js (v14+)
- Python 3.8+
- MongoDB

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
MONGO_URI=mongodb://localhost:27017/enterprise_user_management
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
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

Create `.env` file:
```env
MODEL_PATH=./models
BACKEND_URL=http://localhost:5000
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

Create `.env` file:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Key API Endpoints

### Authentication

```javascript
// Register user
POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer <token>" }

// Create user
POST /api/users
Headers: { Authorization: "Bearer <token>" }
{
  "username": "jane.smith",
  "email": "jane@example.com",
  "role": "manager",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Headers: { Authorization: "Bearer <token>" }
{
  "role": "admin",
  "isActive": true
}

// Delete user
DELETE /api/users/:userId
Headers: { Authorization: "Bearer <token>" }
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks/user/:userId
Headers: { Authorization: "Bearer <token>" }

// Create task
POST /api/tasks
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:taskId/status
Headers: { Authorization: "Bearer <token>" }
{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:taskId/time
Headers: { Authorization: "Bearer <token>" }
{
  "timeSpent": 3600  // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { Authorization: "Bearer <token>" }
{
  "subject": "Login issue",
  "description": "Cannot login with correct credentials",
  "priority": "high",
  "category": "technical"
}

// Get user tickets
GET /api/tickets/user/:userId
Headers: { Authorization: "Bearer <token>" }

// Update ticket
PATCH /api/tickets/:ticketId
Headers: { Authorization: "Bearer <token>" }
{
  "status": "resolved",
  "response": "Password reset link sent"
}
```

## AI/ML Service API

### Risk Prediction

```python
# Python client example
import requests

def predict_user_risk(user_data):
    response = requests.post(
        f"{ML_SERVICE_URL}/api/ml/risk-prediction",
        json={
            "userId": user_data["id"],
            "tasksCompleted": user_data["tasksCompleted"],
            "tasksOverdue": user_data["tasksOverdue"],
            "avgCompletionTime": user_data["avgCompletionTime"],
            "ticketsRaised": user_data["ticketsRaised"]
        }
    )
    return response.json()
    # Returns: { "riskScore": 0.75, "riskLevel": "high", "factors": [...] }
```

### Ticket Classification

```python
def classify_ticket(ticket_text):
    response = requests.post(
        f"{ML_SERVICE_URL}/api/ml/classify-ticket",
        json={
            "subject": "Cannot access dashboard",
            "description": ticket_text
        }
    )
    return response.json()
    # Returns: { "category": "technical", "priority": "high", "confidence": 0.89 }
```

### Anomaly Detection

```python
def detect_anomalies(user_activity):
    response = requests.post(
        f"{ML_SERVICE_URL}/api/ml/anomaly-detection",
        json={
            "userId": user_activity["userId"],
            "loginTimes": user_activity["loginTimes"],
            "activityPattern": user_activity["activityPattern"],
            "accessedResources": user_activity["accessedResources"]
        }
    )
    return response.json()
    # Returns: { "isAnomaly": true, "anomalyScore": 0.92, "reasons": [...] }
```

### Burnout Detection

```python
def detect_burnout(user_metrics):
    response = requests.post(
        f"{ML_SERVICE_URL}/api/ml/burnout-detection",
        json={
            "userId": user_metrics["userId"],
            "workHours": user_metrics["weeklyHours"],
            "tasksAssigned": user_metrics["activeTasks"],
            "completionRate": user_metrics["completionRate"],
            "stressIndicators": user_metrics["ticketFrequency"]
        }
    )
    return response.json()
    # Returns: { "burnoutRisk": "high", "score": 0.85, "recommendations": [...] }
```

## Frontend Integration Patterns

### Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
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
  };

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      // Fetch current user
      axios.get(`${API_URL}/api/auth/me`)
        .then(res => setUser(res.data))
        .catch(() => logout());
    }
  }, [token]);

  return { user, login, logout, isAuthenticated: !!token };
};
```

### Task Board Component

```javascript
// components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await axios.get(`${API_URL}/api/tasks/user/${userId}`);
    const grouped = response.data.reduce((acc, task) => {
      const status = task.status.replace('-', '');
      if (!acc[status]) acc[status] = [];
      acc[status].push(task);
      return acc;
    }, { todo: [], inProgress: [], done: [] });
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await axios.patch(`${API_URL}/api/tasks/${taskId}/status`, {
      status: newStatus
    });
    fetchTasks();
  };

  return (
    <div className="task-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="task-column">
          <h3>{status.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>
                {task.priority}
              </span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const ML_URL = process.env.REACT_APP_ML_URL;

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    const [risk, burnout, anomalies] = await Promise.all([
      axios.post(`${ML_URL}/api/ml/risk-prediction`, { userId }),
      axios.post(`${ML_URL}/api/ml/burnout-detection`, { userId }),
      axios.post(`${ML_URL}/api/ml/anomaly-detection`, { userId })
    ]);

    setAnalytics({
      risk: risk.data,
      burnout: burnout.data,
      anomalies: anomalies.data
    });
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="risk-card">
        <h3>Risk Assessment</h3>
        <div className={`risk-level ${analytics.risk.riskLevel}`}>
          {analytics.risk.riskLevel.toUpperCase()}
        </div>
        <p>Score: {(analytics.risk.riskScore * 100).toFixed(1)}%</p>
      </div>

      <div className="burnout-card">
        <h3>Burnout Detection</h3>
        <div className={`burnout-level ${analytics.burnout.burnoutRisk}`}>
          {analytics.burnout.burnoutRisk.toUpperCase()}
        </div>
        <ul>
          {analytics.burnout.recommendations.map((rec, i) => (
            <li key={i}>{rec}</li>
          ))}
        </ul>
      </div>

      {analytics.anomalies.isAnomaly && (
        <div className="anomaly-alert">
          <h3>⚠️ Anomaly Detected</h3>
          <p>Score: {(analytics.anomalies.anomalyScore * 100).toFixed(1)}%</p>
          <ul>
            {analytics.anomalies.reasons.map((reason, i) => (
              <li key={i}>{reason}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation Patterns

### JWT Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'No authentication token' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.taskId,
      { status: req.body.status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const task = await Task.findById(req.params.taskId);
    task.timeSpent = (task.timeSpent || 0) + req.body.timeSpent;
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

## ML Service Implementation

### FastAPI ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Optional

app = FastAPI()

# Load ML models
risk_model = joblib.load('./models/risk_model.pkl')
burnout_model = joblib.load('./models/burnout_model.pkl')

class RiskPredictionRequest(BaseModel):
    userId: str
    tasksCompleted: int
    tasksOverdue: int
    avgCompletionTime: float
    ticketsRaised: int

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workHours: float
    tasksAssigned: int
    completionRate: float
    stressIndicators: int

@app.post("/api/ml/risk-prediction")
async def predict_risk(request: RiskPredictionRequest):
    features = np.array([[
        request.tasksCompleted,
        request.tasksOverdue,
        request.avgCompletionTime,
        request.ticketsRaised
    ]])
    
    risk_score = risk_model.predict_proba(features)[0][1]
    
    risk_level = "low" if risk_score < 0.3 else "medium" if risk_score < 0.7 else "high"
    
    factors = []
    if request.tasksOverdue > 3:
        factors.append("High number of overdue tasks")
    if request.avgCompletionTime > 7:
        factors.append("Slow task completion")
    if request.ticketsRaised > 5:
        factors.append("Frequent support requests")
    
    return {
        "riskScore": float(risk_score),
        "riskLevel": risk_level,
        "factors": factors
    }

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutDetectionRequest):
    features = np.array([[
        request.workHours,
        request.tasksAssigned,
        request.completionRate,
        request.stressIndicators
    ]])
    
    burnout_score = burnout_model.predict_proba(features)[0][1]
    
    burnout_risk = "low" if burnout_score < 0.4 else "medium" if burnout_score < 0.7 else "high"
    
    recommendations = []
    if request.workHours > 50:
        recommendations.append("Reduce work hours to healthy levels")
    if request.tasksAssigned > 15:
        recommendations.append("Redistribute workload")
    if request.completionRate < 0.6:
        recommendations.append("Provide additional support or training")
    
    return {
        "burnoutRisk": burnout_risk,
        "score": float(burnout_score),
        "recommendations": recommendations
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Configuration

### Database Models

```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'manager', 'admin'], default: 'user' },
  department: String,
  isActive: { type: Boolean, default: true },
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
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

## Troubleshooting

### JWT Token Issues
```javascript
// Verify token is being sent correctly
axios.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

### CORS Errors
```javascript
// backend/server.js
const cors = require('cors');
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Connection Issues
```javascript
// Check ML service health
const checkMLService = async () => {
  try {
    const response = await axios.get(`${process.env.ML_SERVICE_URL}/health`);
    console.log('ML Service:', response.data);
  } catch (error) {
    console.error('ML Service unavailable:', error.message);
  }
};
```

### MongoDB Connection
```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
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
