---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create task tracking with burnout detection"
  - "build support ticket system with AI classification"
  - "add risk prediction to user management"
  - "integrate ML service for anomaly detection"
  - "configure JWT authentication for enterprise app"
  - "deploy user management system with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights. Built with React, Node.js, FastAPI, and MongoDB.

## What This Project Does

The system provides:
- **User Management**: Role-based access control, authentication with JWT
- **Task Tracking**: Kanban boards, time tracking, workload monitoring
- **Support Tickets**: AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout detection, project delay prediction
- **Admin Dashboard**: Organization-wide analytics and audit logs

## Installation

### Prerequisites

```bash
# Node.js 14+ and Python 3.8+ required
node --version
python --version
```

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup (Node.js)

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
# Runs at http://localhost:5000
```

### ML Service Setup (FastAPI)

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=info
EOF

# Start ML service
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
EOF

# Start frontend
npm start
# Runs at http://localhost:3000
```

## Architecture Overview

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   React     │─────▶│   Node.js   │─────▶│   MongoDB   │
│  Frontend   │      │   Backend   │      │  Database   │
└─────────────┘      └─────────────┘      └─────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │   FastAPI   │
                     │ ML Service  │
                     └─────────────┘
```

## Key API Endpoints

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
      role: userData.role || 'user'
    })
  });
  return response.json();
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

### User Management

```javascript
// GET /api/users - Get all users (admin only)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
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
  return response.json();
};

// DELETE /api/users/:id - Delete user (admin only)
const deleteUser = async (userId, token) => {
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
      assignedTo: taskData.assignedTo,
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'in_progress', 'done'
    })
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'todo', 'in_progress', 'done'
  });
  return response.json();
};

// POST /api/tasks/:id/time - Track time
const trackTime = async (taskId, timeData, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      duration: timeData.duration, // in seconds
      date: new Date().toISOString()
    })
  });
  return response.json();
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
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets
const getTickets = async (token, filters = {}) => {
  const params = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tickets?${params}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## ML Service API

### Risk Prediction

```javascript
// POST /predict/risk - Predict user risk level
const predictUserRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/predict/risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userData.userId,
      login_frequency: userData.loginFrequency,
      task_completion_rate: userData.taskCompletionRate,
      failed_login_attempts: userData.failedLoginAttempts,
      avg_session_duration: userData.avgSessionDuration
    })
  });
  const result = await response.json();
  // Returns: { risk_level: 'low' | 'medium' | 'high', confidence: 0.85 }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /detect/anomaly - Detect anomalous behavior
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/detect/anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: activityData.userId,
      login_time: activityData.loginTime,
      ip_address: activityData.ipAddress,
      location: activityData.location,
      activity_pattern: activityData.activityPattern
    })
  });
  const result = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.92, reasons: [...] }
  return result;
};
```

### Burnout Detection

```javascript
// POST /analyze/burnout - Analyze burnout risk
const analyzeBurnout = async (workloadData) => {
  const response = await fetch('http://localhost:8000/analyze/burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: workloadData.userId,
      weekly_hours: workloadData.weeklyHours,
      task_count: workloadData.taskCount,
      overtime_hours: workloadData.overtimeHours,
      missed_deadlines: workloadData.missedDeadlines,
      stress_indicators: workloadData.stressIndicators
    })
  });
  const result = await response.json();
  // Returns: { burnout_risk: 'low' | 'medium' | 'high', recommendations: [...] }
  return result;
};
```

### Ticket Classification

```javascript
// POST /classify/ticket - Classify support ticket
const classifyTicket = async (ticketContent) => {
  const response = await fetch('http://localhost:8000/classify/ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketContent.title,
      description: ticketContent.description
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', suggested_team: 'IT' }
  return result;
};
```

### Project Delay Prediction

```javascript
// POST /predict/delay - Predict project delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict/delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.projectId,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      avg_completion_time: projectData.avgCompletionTime,
      deadline: projectData.deadline,
      team_size: projectData.teamSize
    })
  });
  const result = await response.json();
  // Returns: { delay_probability: 0.75, estimated_delay_days: 5 }
  return result;
};
```

## React Component Patterns

### Protected Route with JWT

```javascript
// components/ProtectedRoute.jsx
import React from 'react';
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

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import ProtectedRoute from './components/ProtectedRoute';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route 
          path="/admin" 
          element={
            <ProtectedRoute requiredRole="admin">
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

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch('http://localhost:5000/api/tasks', {
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

  const moveTask = async (taskId, newStatus) => {
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

  return (
    <div className="kanban-board">
      {['todo', 'in_progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <div className="task-actions">
                {status !== 'todo' && (
                  <button onClick={() => moveTask(task._id, 'todo')}>← Todo</button>
                )}
                {status !== 'in_progress' && (
                  <button onClick={() => moveTask(task._id, 'in_progress')}>
                    In Progress
                  </button>
                )}
                {status !== 'done' && (
                  <button onClick={() => moveTask(task._id, 'done')}>Done →</button>
                )}
              </div>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskLevel: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    
    // Fetch user statistics
    const userStats = await fetch(`http://localhost:5000/api/users/${userId}/stats`, {
      headers: { 'Authorization': `Bearer ${token}` }
    }).then(res => res.json());

    // Get AI predictions
    const [riskPrediction, burnoutAnalysis] = await Promise.all([
      fetch('http://localhost:8000/predict/risk', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          user_id: userId,
          login_frequency: userStats.loginFrequency,
          task_completion_rate: userStats.completionRate,
          failed_login_attempts: userStats.failedLogins,
          avg_session_duration: userStats.avgSessionDuration
        })
      }).then(res => res.json()),
      
      fetch('http://localhost:8000/analyze/burnout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          user_id: userId,
          weekly_hours: userStats.weeklyHours,
          task_count: userStats.taskCount,
          overtime_hours: userStats.overtimeHours,
          missed_deadlines: userStats.missedDeadlines
        })
      }).then(res => res.json())
    ]);

    setAnalytics({
      riskLevel: riskPrediction.risk_level,
      burnoutRisk: burnoutAnalysis.burnout_risk,
      recommendations: burnoutAnalysis.recommendations
    });
  };

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      <div className="metric-card">
        <h3>Risk Level</h3>
        <span className={`badge ${analytics.riskLevel}`}>
          {analytics.riskLevel}
        </span>
      </div>

      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <span className={`badge ${analytics.burnoutRisk}`}>
          {analytics.burnoutRisk}
        </span>
        {analytics.recommendations && (
          <ul>
            {analytics.recommendations.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Backend Patterns (Node.js/Express)

### JWT Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = decoded.userId;
    req.userRole = decoded.role;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.userRole !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
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
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({
      $or: [
        { assignedTo: req.userId },
        { createdBy: req.userId }
      ]
    }).populate('assignedTo', 'username email');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
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
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  profile: {
    firstName: String,
    lastName: String,
    department: String
  },
  stats: {
    loginFrequency: { type: Number, default: 0 },
    taskCompletionRate: { type: Number, default: 0 },
    failedLogins: { type: Number, default: 0 }
  },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date
});

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

userSchema.methods.comparePassword = async function(password) {
  return bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
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
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  dueDate: Date,
  timeTracked: [{ 
    duration: Number, 
    date: Date 
  }],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

## ML Service Implementation (FastAPI)

### Main FastAPI Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from river import anomaly, ensemble
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: float
    task_completion_rate: float
    failed_login_attempts: int
    avg_session_duration: float

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    weekly_hours: float
    task_count: int
    overtime_hours: float
    missed_deadlines: int
    stress_indicators: Optional[List[str]] = []

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

@app.post("/predict/risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Simple rule-based risk scoring
        risk_score = 0
        
        if request.failed_login_attempts > 3:
            risk_score += 30
        if request.task_completion_rate < 0.5:
            risk_score += 25
        if request.login_frequency < 2:
            risk_score += 20
        if request.avg_session_duration > 480:  # > 8 hours
            risk_score += 15
        
        if risk_score >= 60:
            risk_level = "high"
        elif risk_score >= 30:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return {
            "risk_level": risk_level,
            "risk_score": risk_score,
            "confidence": 0.85,
            "factors": {
                "failed_logins": request.failed_login_attempts,
                "completion_rate": request.task_completion_rate
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze/burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    try:
        burnout_score = 0
        recommendations = []
        
        if request.weekly_hours > 50:
            burnout_score += 40
            recommendations.append("Reduce weekly working hours to under 45")
        
        if request.overtime_hours > 10:
            burnout_score += 30
            recommendations.append("Limit overtime to essential tasks only")
        
        if request.missed_deadlines > 2:
            burnout_score += 20
            recommendations.append("Review workload distribution and deadlines")
        
        if request.task_count > 15:
            burnout_score += 10
            recommendations.append("Delegate tasks to other team members")
        
        if burnout_score >= 60:
            burnout_risk = "high"
            recommendations.append("Consider immediate intervention or time off")
        elif burnout_score >= 30:
            burnout_risk = "medium"
            recommendations.append("Monitor workload closely")
        else:
            burnout_risk = "low"
        
        return {
            "burnout_risk": burnout_risk,
            "burnout_score": burnout_score,
            "recommendations": recommendations,
            "stress_indicators": request.stress_indicators
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/classify/ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple keyword-based classification
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            priority = 'high'
            team = 'IT'
        elif any(word in text for word in ['password', 'login', 'access']):
            category = 'access'
            priority = 'medium'
            team = 'IT Support'
        elif any(word in text for word in ['feature', 'request', 'enhancement']):
            category = 'feature_request'
            priority = 'low'
            team = 'Product'
        else:
            category = 'general'
            priority = 'medium'
            team = 'Support'
        
        return {
            "category": category,
            "priority": priority,
            "suggested_team": team,
            "confidence": 0.78
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect/anomaly")
async def detect_anomaly(data: dict):
    try:
        # Simple anomaly detection based on patterns
        anomaly_score = 0
        reasons = []
        
        # Check for unusual login times
        if 'login_time' in data:
            hour = int(data['login_time'].split(':')[0])
            if hour < 6 or hour > 22:
                anomaly_score += 0.3
                reasons.append("Unusual login time")
        
        # Check for multiple failed attempts
        if data.get('failed_attempts', 0) > 3:
            anomaly_score += 0.4
            reasons.append("Multiple failed login attempts")
        
        # Check for location anomaly
        if data.get('new_location', False):
            anomaly_score += 0.3
            reasons.append("Login from new location")
        
        is_anomaly = anomaly_score >= 0.5
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": anomaly_score,
            "reasons": reasons
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/delay")
async def predict_delay(data: dict):
    try:
        total_tasks = data['total_tasks']
        completed_tasks = data['completed_tasks']
        avg_completion_time = data['avg_completion_time']
        
        remaining_tasks = total_tasks - completed_tasks
        estimated_time = remaining_tasks * avg_completion_time
        
        # Simple delay prediction
        if estimated_time > data.get('time_remaining', float('inf')):
            delay_probability = 0.8
            estimated_delay_days = (estimated_time - data.get('time_remaining', 0)) / 24
        else:
            delay_probability = 0.2
            estimated_delay_days = 0
        
        return {
            "delay_probability": delay_probability,
            "estimated_delay_days": int(estimated_delay_days),
            "completion_percentage": (completed_tasks / total_tasks) * 100
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Configuration

### Environment Variables

```bash
# Backend (.env)
PORT=5000
MONGODB_URI=
