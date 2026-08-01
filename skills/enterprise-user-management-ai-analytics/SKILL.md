---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management using React, Node.js, and FastAPI ML service
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - implement JWT authentication with user roles
  - create kanban board with task tracking
  - build AI-powered ticket classification system
  - set up burnout detection and anomaly detection
  - configure MongoDB with user management backend
  - implement risk prediction for enterprise users
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with AI-powered insights. It provides role-based access control (Admin/User), Kanban task boards, support ticket management, and ML-driven features including risk detection, anomaly detection, burnout analysis, and predictive project insights.

**Architecture:**
- **Frontend:** React.js (port 3000)
- **Backend:** Node.js with Express (port 5000)
- **ML Service:** FastAPI with scikit-learn and River (port 8000)
- **Database:** MongoDB
- **Auth:** JWT tokens

## Installation

### Prerequisites

```bash
# Required
node -v  # v14+ recommended
python -v  # 3.8+ for ML service
mongod --version  # MongoDB 4.4+
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
```

Create `backend/.env`:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
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

Create `ml-service/.env`:
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

Create `frontend/.env`:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Core API Endpoints

### Authentication (Backend)

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user' // 'admin' or 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  // Store token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin Only)

```javascript
// GET /api/users - Get all users
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

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
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
      assignedTo: taskData.userId,
      status: 'todo', // 'todo', 'in_progress', 'done'
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
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
    body: JSON.stringify({ status })
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
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets (admin) or user tickets
const getTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

## AI/ML Service API

### Risk Detection

```javascript
// POST /api/ml/risk-detection
const detectUserRisk = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-detection', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      login_frequency: 45,
      failed_logins: 3,
      data_access_count: 150,
      after_hours_access: 8
    })
  });
  const data = await response.json();
  // Returns: { risk_score: 0.75, risk_level: 'high', factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/anomaly-detection
const detectAnomaly = async (behaviorData, token) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: behaviorData.userId,
      features: {
        login_time: behaviorData.loginTime,
        session_duration: behaviorData.sessionDuration,
        actions_per_session: behaviorData.actionsPerSession,
        location_change: behaviorData.locationChange
      }
    })
  });
  const data = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.89, alert_required: true }
  return data;
};
```

### Burnout Detection

```javascript
// POST /api/ml/burnout-detection
const detectBurnout = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      hours_worked_weekly: 55,
      tasks_completed: 25,
      tasks_overdue: 5,
      avg_task_completion_time: 4.5,
      weekend_work_hours: 10
    })
  });
  const data = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.82, recommendations: [...] }
  return data;
};
```

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketContent, token) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketContent.subject,
      description: ticketContent.description
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', assigned_team: 'DevOps' }
  return data;
};
```

### Predictive Project Insights

```javascript
// POST /api/ml/project-prediction
const predictProjectDelay = async (projectData, token) => {
  const response = await fetch('http://localhost:8000/api/ml/project-prediction', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      project_id: projectData.projectId,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      avg_completion_rate: projectData.avgCompletionRate,
      team_size: projectData.teamSize,
      days_remaining: projectData.daysRemaining
    })
  });
  const data = await response.json();
  // Returns: { delay_probability: 0.68, estimated_delay_days: 5, risk_factors: [...] }
  return data;
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
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  // Decode JWT to check role
  const payload = JSON.parse(atob(token.split('.')[1]));
  
  if (requiredRole && payload.role !== requiredRole) {
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
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    // Group by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      in_progress: data.filter(t => t.status === 'in_progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };
  
  const moveTask = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
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
              <select value={task.status} onChange={(e) => moveTask(task._id, e.target.value)}>
                <option value="todo">To Do</option>
                <option value="in_progress">In Progress</option>
                <option value="done">Done</option>
              </select>
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
// components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    highRiskUsers: [],
    burnoutAlerts: [],
    anomalies: []
  });
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    fetchAnalytics();
  }, []);
  
  const fetchAnalytics = async () => {
    // Get all users
    const usersRes = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const users = await usersRes.json();
    
    // Check risk for each user
    const riskPromises = users.map(user => 
      fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/risk-detection`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          user_id: user._id,
          login_frequency: user.loginFrequency || 0,
          failed_logins: user.failedLogins || 0,
          data_access_count: user.dataAccessCount || 0,
          after_hours_access: user.afterHoursAccess || 0
        })
      }).then(r => r.json())
    );
    
    const riskResults = await Promise.all(riskPromises);
    const highRisk = riskResults
      .filter(r => r.risk_level === 'high')
      .map((r, i) => ({ ...users[i], riskScore: r.risk_score }));
    
    setAnalytics({ ...analytics, highRiskUsers: highRisk });
  };
  
  return (
    <div className="analytics-dashboard">
      <h2>AI Analytics Dashboard</h2>
      
      <section className="high-risk-users">
        <h3>High Risk Users ({analytics.highRiskUsers.length})</h3>
        {analytics.highRiskUsers.map(user => (
          <div key={user._id} className="alert-card">
            <p>{user.name} - Risk Score: {(user.riskScore * 100).toFixed(0)}%</p>
          </div>
        ))}
      </section>
    </div>
  );
};

export default AdminAnalytics;
```

## Backend Middleware Examples

### JWT Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'Access denied. No token provided.' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(400).json({ error: 'Invalid token.' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Usage in Routes

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();
const { authMiddleware, adminOnly } = require('../middleware/auth');
const User = require('../models/User');

// Get all users (admin only)
router.get('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update user
router.put('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true }
    ).select('-password');
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## MongoDB Models

### User Model

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  loginFrequency: { type: Number, default: 0 },
  failedLogins: { type: Number, default: 0 },
  dataAccessCount: { type: Number, default: 0 },
  afterHoursAccess: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now }
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

### Task Model

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
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
  dueDate: { type: Date },
  timeTracked: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  completedAt: { type: Date }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  category: { type: String }, // AI-classified
  status: { 
    type: String, 
    enum: ['open', 'in_progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  assignedTeam: { type: String }, // AI-assigned
  createdAt: { type: Date, default: Date.now },
  resolvedAt: { type: Date }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## ML Service Implementation (Python/FastAPI)

### Main FastAPI App

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models (load or initialize)
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class RiskDetectionRequest(BaseModel):
    user_id: str
    login_frequency: int
    failed_logins: int
    data_access_count: int
    after_hours_access: int

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    hours_worked_weekly: float
    tasks_completed: int
    tasks_overdue: int
    avg_task_completion_time: float
    weekend_work_hours: float

@app.post("/api/ml/risk-detection")
async def detect_risk(request: RiskDetectionRequest):
    # Simple risk scoring algorithm
    features = np.array([
        request.login_frequency,
        request.failed_logins,
        request.data_access_count,
        request.after_hours_access
    ])
    
    # Normalize and calculate risk score
    weights = np.array([0.1, 0.4, 0.3, 0.2])
    normalized = features / np.array([100, 10, 500, 50])  # Max expected values
    risk_score = min(1.0, np.dot(normalized, weights))
    
    if risk_score > 0.7:
        risk_level = "high"
    elif risk_score > 0.4:
        risk_level = "medium"
    else:
        risk_level = "low"
    
    factors = []
    if request.failed_logins > 5:
        factors.append("High failed login attempts")
    if request.after_hours_access > 20:
        factors.append("Excessive after-hours access")
    
    return {
        "user_id": request.user_id,
        "risk_score": float(risk_score),
        "risk_level": risk_level,
        "factors": factors
    }

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutDetectionRequest):
    # Burnout scoring
    overwork_score = min(1.0, request.hours_worked_weekly / 60)
    overdue_ratio = request.tasks_overdue / max(1, request.tasks_completed)
    weekend_score = min(1.0, request.weekend_work_hours / 20)
    
    burnout_score = (overwork_score * 0.4 + 
                     min(1.0, overdue_ratio) * 0.3 + 
                     weekend_score * 0.3)
    
    if burnout_score > 0.7:
        risk = "high"
        recommendations = [
            "Reduce weekly hours",
            "Redistribute tasks",
            "Schedule time off"
        ]
    elif burnout_score > 0.4:
        risk = "medium"
        recommendations = [
            "Monitor workload",
            "Avoid weekend work"
        ]
    else:
        risk = "low"
        recommendations = []
    
    return {
        "user_id": request.user_id,
        "burnout_risk": risk,
        "score": float(burnout_score),
        "recommendations": recommendations
    }

@app.post("/api/ml/classify-ticket")
async def classify_ticket(subject: str, description: str):
    # Simple keyword-based classification
    text = (subject + " " + description).lower()
    
    if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
        category = "technical"
        priority = "high"
        assigned_team = "DevOps"
    elif any(word in text for word in ['access', 'permission', 'login']):
        category = "access"
        priority = "medium"
        assigned_team = "IT Support"
    elif any(word in text for word in ['feature', 'request', 'enhancement']):
        category = "feature_request"
        priority = "low"
        assigned_team = "Product"
    else:
        category = "general"
        priority = "medium"
        assigned_team = "Support"
    
    return {
        "category": category,
        "priority": priority,
        "assigned_team": assigned_team
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Configuration Files

### Backend package.json

```json
{
  "name": "enterprise-user-mgmt-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mongoose": "^7.0.0",
    "jsonwebtoken": "^9.0.0",
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "dotenv": "^16.0.3"
  },
  "devDependencies": {
    "nodemon": "^2.0.20"
  }
}
```

### ML Service requirements.txt

```txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.4.2
numpy==1.24.3
scikit-learn==1.3.0
river==0.18.0
joblib==1.3.2
python-dotenv==1.0.0
```

## Common Workflows

### Complete User Registration and Task Assignment Flow

```javascript
// 1. Admin registers a new user
const newUser = await registerUser({
  name: "John Doe",
  email: "john@example.com",
  password: "securePassword123",
  role: "user"
});

// 2. Admin assigns task to user
const token = localStorage.getItem('token');
const task = await createTask({
  title: "Complete project documentation",
  description: "Write comprehensive docs",
  userId: newUser._id,
  priority: "high",
  dueDate: "2026-05-01"
}, token);

// 3. Check user for burnout risk
const burnoutCheck = await detectBurnout(newUser._id, token);
if (burnoutCheck.burnout_risk === 'high') {
  console.warn("User at risk of burnout:", burnoutCheck.recommendations);
}
```

### Ticket Creation with AI Classification

```javascript
const createAndClassifyTicket = async (ticketData, token) => {
  // 1. Classify ticket using ML service
  const classification = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/classify-ticket`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description
    })
  }).then(r => r.json());
  
  // 2. Create ticket with AI classification
  const ticket = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      ...ticketData,
      category: classification.category,
      priority: classification.priority,
      assignedTeam: classification.assigned_team
    })
  }).then(r => r.json());
  
  return ticket;
};
```

## Troubleshooting

### JWT Token Issues

```javascript
// Verify token is valid
const verifyToken = (token) => {
  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    const isExpired = payload.exp * 1000 < Date.now();
    if (isExpired) {
      console.error("Token expired");
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return payload;
  } catch (e) {
    console.error("Invalid token format");
    return null;
  }
};
```

### MongoDB Connection Issues

```javascript
// backend/server.js
const mongoose = require('mongoose');

mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
})
.then(() => console.log('MongoDB connected'))
.
