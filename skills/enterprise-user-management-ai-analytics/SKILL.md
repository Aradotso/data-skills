---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management capabilities
triggers:
  - "help me integrate the enterprise user management system"
  - "how do I set up AI analytics for user management"
  - "show me how to use the user management API"
  - "implement task tracking with AI insights"
  - "configure the enterprise user management system"
  - "integrate AI-powered ticket classification"
  - "set up user authentication with JWT in this system"
  - "add anomaly detection to user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System with AI Analytics is a full-stack application that provides centralized user, task, and support ticket management with integrated AI capabilities. It offers:

- User authentication and role-based access control (RBAC)
- Task management with Kanban boards and time tracking
- Support ticket system with AI classification
- AI-powered analytics: risk detection, anomaly detection, burnout analysis, and predictive insights
- Admin dashboard for user and organization management
- ML service for real-time predictions using scikit-learn and River

## Installation

### Prerequisites

- Node.js (v14+)
- Python (v3.8+)
- MongoDB (running instance)

### Clone and Setup

```bash
# Clone the repository
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

**Backend (.env)**
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend (.env)**
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

**ML Service (.env)**
```bash
# ml-service/.env
MODEL_PATH=./models
LOG_LEVEL=INFO
```

### Starting Services

```bash
# Terminal 1 - Backend
cd backend
npm start

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload --port 8000

# Terminal 3 - Frontend
cd frontend
npm start
```

## Key API Endpoints

### Authentication

**Register User**
```javascript
// POST /api/auth/register
const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/register`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    username: 'john_doe',
    email: 'john@example.com',
    password: 'SecurePass123!',
    role: 'user' // 'user' or 'admin'
  })
});
const data = await response.json();
// Returns: { token, user: { id, username, email, role } }
```

**Login**
```javascript
// POST /api/auth/login
const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/login`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'john@example.com',
    password: 'SecurePass123!'
  })
});
const { token, user } = await response.json();
localStorage.setItem('token', token);
```

### User Management (Admin)

**Get All Users**
```javascript
// GET /api/users
const token = localStorage.getItem('token');
const response = await fetch(`${process.env.REACT_APP_API_URL}/users`, {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});
const users = await response.json();
```

**Update User**
```javascript
// PUT /api/users/:id
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      username: updates.username,
      email: updates.email,
      role: updates.role,
      status: updates.status // 'active', 'inactive', 'suspended'
    })
  });
  return await response.json();
};
```

**Delete User**
```javascript
// DELETE /api/users/:id
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  await fetch(`${process.env.REACT_APP_API_URL}/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
};
```

### Task Management

**Create Task**
```javascript
// POST /api/tasks
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: 'high', // 'low', 'medium', 'high', 'critical'
      status: 'todo', // 'todo', 'in_progress', 'done'
      dueDate: taskData.dueDate,
      estimatedHours: taskData.estimatedHours
    })
  });
  return await response.json();
};
```

**Get User Tasks**
```javascript
// GET /api/tasks/user/:userId
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:taskId/status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return await response.json();
};
```

**Track Time**
```javascript
// POST /api/tasks/:taskId/time
const trackTime = async (taskId, timeData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      duration: timeData.duration, // in minutes
      description: timeData.description,
      date: new Date().toISOString()
    })
  });
  return await response.json();
};
```

### Support Tickets

**Create Ticket**
```javascript
// POST /api/tickets
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tickets`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category, // 'technical', 'billing', 'general'
      priority: 'medium'
    })
  });
  return await response.json();
};
```

**Get Tickets**
```javascript
// GET /api/tickets
const getTickets = async (filters = {}) => {
  const token = localStorage.getItem('token');
  const queryParams = new URLSearchParams(filters).toString();
  const response = await fetch(
    `${process.env.REACT_APP_API_URL}/tickets?${queryParams}`,
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return await response.json();
};
```

## AI/ML Service API

### AI-Powered Ticket Classification

```javascript
// POST /classify-ticket (ML Service)
const classifyTicket = async (ticketText) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/classify-ticket`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      title: 'Issue with login'
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.89 }
  return result;
};
```

### Risk Prediction

```javascript
// POST /predict-risk (ML Service)
const predictUserRisk = async (userData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict-risk`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.userId,
      loginAttempts: userData.loginAttempts,
      failedLogins: userData.failedLogins,
      lastLoginTime: userData.lastLoginTime,
      accessPatterns: userData.accessPatterns,
      suspiciousActivities: userData.suspiciousActivities
    })
  });
  const result = await response.json();
  // Returns: { riskScore: 0.75, riskLevel: 'high', factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /detect-anomaly (ML Service)
const detectAnomaly = async (activityData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/detect-anomaly`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: activityData.userId,
      timestamp: new Date().toISOString(),
      action: activityData.action,
      ipAddress: activityData.ipAddress,
      location: activityData.location,
      deviceInfo: activityData.deviceInfo
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.82, reason: 'Unusual login location' }
  return result;
};
```

### Burnout Detection

```javascript
// POST /detect-burnout (ML Service)
const detectBurnout = async (userId) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/detect-burnout`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      hoursWorked: 55, // weekly hours
      tasksCompleted: 12,
      missedDeadlines: 3,
      workloadTrend: 'increasing'
    })
  });
  const result = await response.json();
  // Returns: { burnoutRisk: 'high', score: 0.78, recommendations: [...] }
  return result;
};
```

### Predictive Project Insights

```javascript
// POST /predict-project-delay (ML Service)
const predictProjectDelay = async (projectData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict-project-delay`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.projectId,
      tasksTotal: projectData.tasksTotal,
      tasksCompleted: projectData.tasksCompleted,
      daysRemaining: projectData.daysRemaining,
      teamSize: projectData.teamSize,
      complexity: projectData.complexity
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.65, estimatedDelay: 5, factors: [...] }
  return result;
};
```

## Common Patterns

### Protected Route Component (React)

```javascript
// components/ProtectedRoute.js
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
```

### API Client with Auth

```javascript
// utils/apiClient.js
const apiClient = {
  async request(url, options = {}) {
    const token = localStorage.getItem('token');
    const headers = {
      'Content-Type': 'application/json',
      ...options.headers
    };

    if (token) {
      headers['Authorization'] = `Bearer ${token}`;
    }

    const response = await fetch(url, {
      ...options,
      headers
    });

    if (response.status === 401) {
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      window.location.href = '/login';
      throw new Error('Unauthorized');
    }

    return response.json();
  },

  get(url) {
    return this.request(url);
  },

  post(url, data) {
    return this.request(url, {
      method: 'POST',
      body: JSON.stringify(data)
    });
  },

  put(url, data) {
    return this.request(url, {
      method: 'PUT',
      body: JSON.stringify(data)
    });
  },

  delete(url) {
    return this.request(url, { method: 'DELETE' });
  }
};

export default apiClient;
```

### Kanban Board Implementation

```javascript
// components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import apiClient from '../utils/apiClient';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });

  useEffect(() => {
    loadTasks();
  }, [userId]);

  const loadTasks = async () => {
    const allTasks = await apiClient.get(
      `${process.env.REACT_APP_API_URL}/tasks/user/${userId}`
    );
    
    const grouped = allTasks.reduce((acc, task) => {
      acc[task.status] = acc[task.status] || [];
      acc[task.status].push(task);
      return acc;
    }, { todo: [], in_progress: [], done: [] });
    
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await apiClient.patch(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
      { status: newStatus }
    );
    loadTasks();
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
              <button onClick={() => moveTask(task._id, getNextStatus(status))}>
                Move →
              </button>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

const getNextStatus = (currentStatus) => {
  const statusFlow = { todo: 'in_progress', in_progress: 'done', done: 'done' };
  return statusFlow[currentStatus];
};

export default KanbanBoard;
```

### Real-time AI Integration

```javascript
// services/aiService.js
const aiService = {
  async analyzeTicket(ticket) {
    // Classify ticket using AI
    const classification = await fetch(
      `${process.env.REACT_APP_ML_URL}/classify-ticket`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          text: ticket.description,
          title: ticket.title
        })
      }
    ).then(res => res.json());

    return {
      ...ticket,
      aiCategory: classification.category,
      aiPriority: classification.priority,
      confidence: classification.confidence
    };
  },

  async checkUserSecurity(userId) {
    const user = await apiClient.get(
      `${process.env.REACT_APP_API_URL}/users/${userId}`
    );

    // Check for anomalies
    const anomalyCheck = await fetch(
      `${process.env.REACT_APP_ML_URL}/detect-anomaly`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          userId: user._id,
          timestamp: new Date().toISOString(),
          action: 'login',
          ipAddress: user.lastLoginIP,
          location: user.lastLoginLocation
        })
      }
    ).then(res => res.json());

    // Predict risk
    const riskPrediction = await fetch(
      `${process.env.REACT_APP_ML_URL}/predict-risk`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          userId: user._id,
          loginAttempts: user.loginAttempts || 0,
          failedLogins: user.failedLogins || 0,
          lastLoginTime: user.lastLogin,
          accessPatterns: user.accessPatterns || []
        })
      }
    ).then(res => res.json());

    return {
      isAnomaly: anomalyCheck.isAnomaly,
      riskLevel: riskPrediction.riskLevel,
      recommendations: [...(anomalyCheck.recommendations || []), ...(riskPrediction.factors || [])]
    };
  }
};

export default aiService;
```

### Admin Dashboard Analytics

```javascript
// components/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import apiClient from '../utils/apiClient';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeUsers: 0,
    totalTasks: 0,
    openTickets: 0,
    highRiskUsers: []
  });

  useEffect(() => {
    loadAnalytics();
  }, []);

  const loadAnalytics = async () => {
    const [users, tasks, tickets] = await Promise.all([
      apiClient.get(`${process.env.REACT_APP_API_URL}/users`),
      apiClient.get(`${process.env.REACT_APP_API_URL}/tasks`),
      apiClient.get(`${process.env.REACT_APP_API_URL}/tickets`)
    ]);

    // Check for high-risk users using AI
    const riskChecks = await Promise.all(
      users.slice(0, 10).map(async (user) => {
        const risk = await fetch(`${process.env.REACT_APP_ML_URL}/predict-risk`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            userId: user._id,
            loginAttempts: user.loginAttempts || 0,
            failedLogins: user.failedLogins || 0
          })
        }).then(res => res.json());
        
        return { user, riskScore: risk.riskScore };
      })
    );

    const highRiskUsers = riskChecks
      .filter(r => r.riskScore > 0.7)
      .map(r => r.user);

    setAnalytics({
      totalUsers: users.length,
      activeUsers: users.filter(u => u.status === 'active').length,
      totalTasks: tasks.length,
      openTickets: tickets.filter(t => t.status === 'open').length,
      highRiskUsers
    });
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p>{analytics.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Active Users</h3>
          <p>{analytics.activeUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Total Tasks</h3>
          <p>{analytics.totalTasks}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p>{analytics.openTickets}</p>
        </div>
      </div>
      
      {analytics.highRiskUsers.length > 0 && (
        <div className="alert-section">
          <h3>⚠️ High Risk Users</h3>
          <ul>
            {analytics.highRiskUsers.map(user => (
              <li key={user._id}>{user.username} - {user.email}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AdminDashboard;
```

## Backend Implementation Examples

### JWT Middleware (Node.js)

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### User Controller

```javascript
// controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

exports.register = async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }

    const user = new User({ username, email, password, role: role || 'user' });
    await user.save();

    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );

    res.status(201).json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    const user = await User.findByIdAndUpdate(id, updates, { new: true }).select('-password');
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.deleteUser = async (req, res) => {
  try {
    await User.findByIdAndDelete(req.params.id);
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

### Task Routes

```javascript
// routes/taskRoutes.js
const express = require('express');
const router = express.Router();
const { authMiddleware, adminOnly } = require('../middleware/auth');
const taskController = require('../controllers/taskController');

router.post('/', authMiddleware, adminOnly, taskController.createTask);
router.get('/', authMiddleware, taskController.getAllTasks);
router.get('/user/:userId', authMiddleware, taskController.getUserTasks);
router.patch('/:taskId/status', authMiddleware, taskController.updateTaskStatus);
router.post('/:taskId/time', authMiddleware, taskController.trackTime);
router.delete('/:taskId', authMiddleware, adminOnly, taskController.deleteTask);

module.exports = router;
```

## ML Service Implementation (Python/FastAPI)

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from river import anomaly, drift

app = FastAPI(title="Enterprise User Management AI Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
class TicketClassificationRequest(BaseModel):
    text: str
    title: str

class RiskPredictionRequest(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    lastLoginTime: Optional[str]
    accessPatterns: Optional[List[dict]]
    suspiciousActivities: Optional[int] = 0

class AnomalyDetectionRequest(BaseModel):
    userId: str
    timestamp: str
    action: str
    ipAddress: str
    location: Optional[str]
    deviceInfo: Optional[dict]

class BurnoutDetectionRequest(BaseModel):
    userId: str
    hoursWorked: float
    tasksCompleted: int
    missedDeadlines: int
    workloadTrend: str

# Load or initialize models
try:
    ticket_classifier = joblib.load('models/ticket_classifier.pkl')
except:
    ticket_classifier = None

anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8)

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        # Simple rule-based classification (replace with trained model)
        text_lower = (request.text + " " + request.title).lower()
        
        if any(word in text_lower for word in ['login', 'password', 'access', 'error', 'bug']):
            category = 'technical'
            priority = 'high'
        elif any(word in text_lower for word in ['payment', 'invoice', 'billing', 'charge']):
            category = 'billing'
            priority = 'medium'
        else:
            category = 'general'
            priority = 'low'
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Calculate risk score based on multiple factors
        risk_score = 0.0
        factors = []
        
        # Failed login attempts
        if request.failedLogins > 3:
            risk_score += 0.3
            factors.append("Multiple failed login attempts")
        
        # Login attempts
        if request.loginAttempts > 10:
            risk_score += 0.2
            factors.append("Unusual number of login attempts")
        
        # Suspicious activities
        if request.suspiciousActivities > 0:
            risk_score += 0.4
            factors.append("Suspicious activities detected")
        
        # Access patterns anomalies
        if request.accessPatterns and len(request.accessPatterns) > 5:
            risk_score += 0.1
            factors.append("Irregular access patterns")
        
        risk_score = min(risk_score, 1.0)
        
        if risk_score >= 0.7:
            risk_level = 'high'
        elif risk_score >= 0.4:
            risk_level = 'medium'
        else:
            risk_level = 'low'
        
        return {
            "riskScore": round(risk_score, 2),
            "riskLevel": risk_level,
            "factors": factors
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        # Create feature vector
        features = {
            'hour': int(request.timestamp[11:13]),
            'action_hash': hash(request.action) % 1000,
            'ip_hash': hash(request.ipAddress) % 1000
        }
        
        # Score the instance
        score = anomaly_detector.score_one(features)
        
        # Update
