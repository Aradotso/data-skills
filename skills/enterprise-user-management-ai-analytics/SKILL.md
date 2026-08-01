---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task automation built with React, Node.js, and FastAPI ML services
triggers:
  - "set up enterprise user management with AI"
  - "implement user task tracking with analytics"
  - "add AI risk detection to user management"
  - "create admin dashboard with ML insights"
  - "build ticket classification system"
  - "configure JWT authentication for enterprise app"
  - "integrate burnout detection analytics"
  - "deploy user management system with AI"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines user/task management with AI-powered analytics. It provides role-based access control, Kanban task tracking, support ticketing, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and project delay forecasting.

**Stack**: React.js frontend, Node.js/Express backend, FastAPI ML service, MongoDB database, JWT authentication

**Key Capabilities**:
- User CRUD operations with role-based permissions (Admin/User)
- Task management with Kanban board and time tracking
- Support ticket system with AI classification
- ML models for risk scoring, anomaly detection, and burnout prediction
- Real-time analytics dashboard

## Installation

### Prerequisites

```bash
# Ensure you have installed
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
```

### Complete Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt

# Install frontend dependencies
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env)**:
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**:
```bash
# ml-service/.env
PORT=8000
MODEL_PATH=./models
BACKEND_URL=http://localhost:5000
LOG_LEVEL=info
```

**Frontend (.env)**:
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

## Running the System

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start
# Running at http://localhost:5000

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload --port 8000
# Running at http://localhost:8000

# Terminal 3 - Frontend
cd frontend
npm start
# Running at http://localhost:3000
```

### Production Build

```bash
# Frontend production build
cd frontend
npm run build

# Backend production
cd backend
NODE_ENV=production npm start

# ML service production
cd ml-service
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

## Backend API Reference

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securepass123"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "user_id",
    "username": "john.doe",
    "role": "user"
  }
}
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Authorization: Bearer {token}

// Create user
POST /api/users
Authorization: Bearer {token}
{
  "username": "jane.smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Authorization: Bearer {token}
{
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:userId
Authorization: Bearer {token}
```

### Task Management

```javascript
// Create task
POST /api/tasks
Authorization: Bearer {token}
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo" // todo, in_progress, done
}

// Update task status
PATCH /api/tasks/:taskId
{
  "status": "in_progress"
}

// Get user tasks
GET /api/tasks/user/:userId
Authorization: Bearer {token}

// Track time
POST /api/tasks/:taskId/time
{
  "duration": 3600 // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer {token}
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error",
  "priority": "medium",
  "category": "technical"
}

// Get all tickets (Admin)
GET /api/tickets
Authorization: Bearer {token}

// Update ticket
PATCH /api/tickets/:ticketId
{
  "status": "resolved",
  "resolution": "Fixed permissions issue"
}
```

## ML Service API

### Risk Prediction

```javascript
// Predict user risk score
POST /api/ml/predict-risk
Content-Type: application/json

{
  "userId": "user_id",
  "features": {
    "loginFailures": 3,
    "taskCompletionRate": 0.65,
    "averageTaskTime": 4.5,
    "ticketCount": 12,
    "workloadScore": 75
  }
}

// Response
{
  "riskScore": 0.72,
  "riskLevel": "high", // low, medium, high
  "factors": ["high_login_failures", "low_completion_rate"]
}
```

### Anomaly Detection

```javascript
// Detect anomalous behavior
POST /api/ml/detect-anomaly
{
  "userId": "user_id",
  "behavior": {
    "loginTime": "2026-04-15T03:30:00Z",
    "loginLocation": "192.168.1.100",
    "actionsPerHour": 150,
    "dataAccess": ["sensitive_document"]
  }
}

// Response
{
  "isAnomaly": true,
  "anomalyScore": 0.89,
  "reasons": ["unusual_time", "high_activity_rate"]
}
```

### Burnout Detection

```javascript
// Analyze burnout risk
POST /api/ml/burnout-analysis
{
  "userId": "user_id",
  "metrics": {
    "weeklyHours": 55,
    "overtimeHours": 15,
    "tasksInProgress": 12,
    "missedDeadlines": 4,
    "avgStressLevel": 7.5
  }
}

// Response
{
  "burnoutRisk": "high",
  "burnoutScore": 0.81,
  "recommendations": [
    "Reduce task load by 30%",
    "Schedule mandatory break",
    "Reassign low-priority tasks"
  ]
}
```

### Ticket Classification

```javascript
// Auto-classify support ticket
POST /api/ml/classify-ticket
{
  "ticketText": "I cannot log in to the system. Getting authentication error.",
  "metadata": {
    "userId": "user_id",
    "timestamp": "2026-04-15T10:00:00Z"
  }
}

// Response
{
  "category": "authentication",
  "priority": "high",
  "assignTo": "security_team",
  "confidence": 0.94
}
```

### Project Delay Prediction

```javascript
// Predict project completion delay
POST /api/ml/predict-delay
{
  "projectId": "proj_123",
  "features": {
    "tasksTotal": 50,
    "tasksCompleted": 20,
    "daysRemaining": 15,
    "teamSize": 5,
    "avgVelocity": 2.3,
    "complexityScore": 8
  }
}

// Response
{
  "predictedDelay": 7, // days
  "completionProbability": 0.45,
  "recommendations": [
    "Add 2 team members",
    "Reduce scope by 10%"
  ]
}
```

## Frontend Integration

### Authentication Setup

```javascript
// src/services/authService.js
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
```

### API Client with JWT

```javascript
// src/services/apiClient.js
import axios from 'axios';
import { authService } from './authService';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add JWT token to requests
apiClient.interceptors.request.use(
  (config) => {
    const token = authService.getToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Handle 401 responses
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      authService.logout();
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### User Service

```javascript
// src/services/userService.js
import apiClient from './apiClient';

export const userService = {
  async getAllUsers() {
    const response = await apiClient.get('/users');
    return response.data;
  },
  
  async createUser(userData) {
    const response = await apiClient.post('/users', userData);
    return response.data;
  },
  
  async updateUser(userId, updates) {
    const response = await apiClient.put(`/users/${userId}`, updates);
    return response.data;
  },
  
  async deleteUser(userId) {
    await apiClient.delete(`/users/${userId}`);
  },
  
  async getUserAnalytics(userId) {
    const response = await apiClient.get(`/users/${userId}/analytics`);
    return response.data;
  }
};
```

### Task Management Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import apiClient from '../services/apiClient';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    in_progress: [],
    done: []
  });
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    try {
      const response = await apiClient.get('/tasks/user/me');
      const grouped = groupTasksByStatus(response.data);
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };
  
  const groupTasksByStatus = (taskList) => {
    return taskList.reduce((acc, task) => {
      if (!acc[task.status]) acc[task.status] = [];
      acc[task.status].push(task);
      return acc;
    }, { todo: [], in_progress: [], done: [] });
  };
  
  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await apiClient.patch(`/tasks/${taskId}`, { status: newStatus });
      fetchTasks(); // Reload tasks
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };
  
  return (
    <div className="task-board">
      {Object.entries(tasks).map(([status, taskList]) => (
        <div key={status} className="task-column">
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          {taskList.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status}
                onChange={(e) => updateTaskStatus(task._id, e.target.value)}
              >
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

export default TaskBoard;
```

### ML Analytics Integration

```javascript
// src/services/mlService.js
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_URL;

export const mlService = {
  async getRiskAnalysis(userId, features) {
    const response = await axios.post(`${ML_API_URL}/api/ml/predict-risk`, {
      userId,
      features
    });
    return response.data;
  },
  
  async detectBurnout(userId, metrics) {
    const response = await axios.post(`${ML_API_URL}/api/ml/burnout-analysis`, {
      userId,
      metrics
    });
    return response.data;
  },
  
  async classifyTicket(ticketText, metadata) {
    const response = await axios.post(`${ML_API_URL}/api/ml/classify-ticket`, {
      ticketText,
      metadata
    });
    return response.data;
  },
  
  async predictProjectDelay(projectId, features) {
    const response = await axios.post(`${ML_API_URL}/api/ml/predict-delay`, {
      projectId,
      features
    });
    return response.data;
  }
};
```

### Admin Analytics Dashboard

```javascript
// src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { userService } from '../services/userService';
import { mlService } from '../services/mlService';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeUsers: 0,
    highRiskUsers: [],
    burnoutAlerts: []
  });
  
  useEffect(() => {
    loadAnalytics();
  }, []);
  
  const loadAnalytics = async () => {
    try {
      const users = await userService.getAllUsers();
      
      // Analyze each user for risk
      const riskAnalyses = await Promise.all(
        users.map(async (user) => {
          const features = {
            loginFailures: user.loginFailures || 0,
            taskCompletionRate: user.taskCompletionRate || 0.8,
            averageTaskTime: user.avgTaskTime || 3,
            ticketCount: user.ticketCount || 0,
            workloadScore: user.workloadScore || 50
          };
          
          const risk = await mlService.getRiskAnalysis(user._id, features);
          return { user, risk };
        })
      );
      
      const highRisk = riskAnalyses.filter(
        (r) => r.risk.riskLevel === 'high'
      );
      
      setAnalytics({
        totalUsers: users.length,
        activeUsers: users.filter(u => u.status === 'active').length,
        highRiskUsers: highRisk,
        burnoutAlerts: [] // Load separately
      });
    } catch (error) {
      console.error('Failed to load analytics:', error);
    }
  };
  
  return (
    <div className="admin-dashboard">
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-value">{analytics.totalUsers}</p>
        </div>
        
        <div className="stat-card">
          <h3>Active Users</h3>
          <p className="stat-value">{analytics.activeUsers}</p>
        </div>
        
        <div className="stat-card alert">
          <h3>High Risk Users</h3>
          <p className="stat-value">{analytics.highRiskUsers.length}</p>
        </div>
      </div>
      
      {analytics.highRiskUsers.length > 0 && (
        <div className="risk-alerts">
          <h3>Risk Alerts</h3>
          {analytics.highRiskUsers.map(({ user, risk }) => (
            <div key={user._id} className="alert-item">
              <strong>{user.username}</strong>
              <span>Risk Score: {(risk.riskScore * 100).toFixed(0)}%</span>
              <ul>
                {risk.factors.map((factor, idx) => (
                  <li key={idx}>{factor}</li>
                ))}
              </ul>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

export default AdminDashboard;
```

## Backend Implementation Patterns

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.createUser = async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = new User({ username, email, password, role });
    await user.save();
    
    res.status(201).json({
      message: 'User created successfully',
      user: { id: user._id, username: user.username, email: user.email }
    });
  } catch (error) {
    res.status(500).json({ message: 'Failed to create user', error: error.message });
  }
};

exports.updateUser = async (req, res) => {
  try {
    const { userId } = req.params;
    const updates = req.body;
    
    const user = await User.findByIdAndUpdate(
      userId,
      { $set: updates },
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: 'Failed to update user', error: error.message });
  }
};

exports.deleteUser = async (req, res) => {
  try {
    const { userId } = req.params;
    
    const user = await User.findByIdAndDelete(userId);
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: 'Failed to delete user', error: error.message });
  }
};
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  try {
    let token;
    
    if (req.headers.authorization?.startsWith('Bearer')) {
      token = req.headers.authorization.split(' ')[1];
    }
    
    if (!token) {
      return res.status(401).json({ message: 'Not authorized, no token' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    
    if (!req.user) {
      return res.status(401).json({ message: 'User not found' });
    }
    
    next();
  } catch (error) {
    res.status(401).json({ message: 'Not authorized, token failed' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        message: `User role ${req.user.role} is not authorized`
      });
    }
    next();
  };
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
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
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
  dueDate: {
    type: Date
  },
  timeSpent: {
    type: Number,
    default: 0 // in seconds
  },
  tags: [String]
}, {
  timestamps: true
});

module.exports = mongoose.model('Task', taskSchema);
```

### Route Configuration

```javascript
// backend/routes/index.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');

// Auth routes
const authController = require('../controllers/authController');
router.post('/auth/register', authController.register);
router.post('/auth/login', authController.login);

// User routes (protected)
const userController = require('../controllers/userController');
router.get('/users', protect, authorize('admin'), userController.getAllUsers);
router.post('/users', protect, authorize('admin'), userController.createUser);
router.put('/users/:userId', protect, authorize('admin'), userController.updateUser);
router.delete('/users/:userId', protect, authorize('admin'), userController.deleteUser);

// Task routes
const taskController = require('../controllers/taskController');
router.get('/tasks/user/me', protect, taskController.getMyTasks);
router.post('/tasks', protect, taskController.createTask);
router.patch('/tasks/:taskId', protect, taskController.updateTask);
router.post('/tasks/:taskId/time', protect, taskController.trackTime);

// Ticket routes
const ticketController = require('../controllers/ticketController');
router.get('/tickets', protect, ticketController.getAllTickets);
router.post('/tickets', protect, ticketController.createTicket);
router.patch('/tickets/:ticketId', protect, ticketController.updateTicket);

module.exports = router;
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

app = FastAPI(title="Enterprise UMS ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class RiskFeatures(BaseModel):
    userId: str
    features: Dict[str, float]

class RiskPrediction(BaseModel):
    riskScore: float
    riskLevel: str
    factors: List[str]

@app.post("/api/ml/predict-risk", response_model=RiskPrediction)
async def predict_risk(data: RiskFeatures):
    try:
        features = data.features
        
        # Calculate risk score
        risk_score = 0.0
        risk_factors = []
        
        # Login failures factor
        if features.get('loginFailures', 0) > 2:
            risk_score += 0.3
            risk_factors.append('high_login_failures')
        
        # Task completion rate
        if features.get('taskCompletionRate', 1.0) < 0.7:
            risk_score += 0.25
            risk_factors.append('low_completion_rate')
        
        # Workload score
        if features.get('workloadScore', 50) > 80:
            risk_score += 0.2
            risk_factors.append('high_workload')
        
        # Ticket count
        if features.get('ticketCount', 0) > 10:
            risk_score += 0.15
            risk_factors.append('high_ticket_volume')
        
        # Determine risk level
        if risk_score < 0.3:
            risk_level = 'low'
        elif risk_score < 0.6:
            risk_level = 'medium'
        else:
            risk_level = 'high'
        
        return RiskPrediction(
            riskScore=min(risk_score, 1.0),
            riskLevel=risk_level,
            factors=risk_factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class BurnoutMetrics(BaseModel):
    userId: str
    metrics: Dict[str, float]

class BurnoutAnalysis(BaseModel):
    burnoutRisk: str
    burnoutScore: float
    recommendations: List[str]

@app.post("/api/ml/burnout-analysis", response_model=BurnoutAnalysis)
async def analyze_burnout(data: BurnoutMetrics):
    try:
        metrics = data.metrics
        
        score = 0.0
        recommendations = []
        
        # Weekly hours
        weekly_hours = metrics.get('weeklyHours', 40)
        if weekly_hours > 50:
            score += 0.3
            recommendations.append('Reduce task load by 30%')
        
        # Overtime
        overtime = metrics.get('overtimeHours', 0)
        if overtime > 10:
            score += 0.25
            recommendations.append('Schedule mandatory break')
        
        # Tasks in progress
        tasks_in_progress = metrics.get('tasksInProgress', 0)
        if tasks_in_progress > 10:
            score += 0.2
            recommendations.append('Reassign low-priority tasks')
        
        # Missed deadlines
        missed = metrics.get('missedDeadlines', 0)
        if missed > 3:
            score += 0.25
            recommendations.append('Extend project timelines')
        
        if score < 0.3:
            risk = 'low'
        elif score < 0.6:
            risk = 'medium'
        else:
            risk = 'high'
        
        return BurnoutAnalysis(
            burnoutRisk=risk,
            burnoutScore=min(score, 1.0),
            recommendations=recommendations if recommendations else ['Continue current workload']
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class TicketData(BaseModel):
    ticketText: str
    metadata: Dict

class TicketClassification(BaseModel):
    category: str
    priority: str
    assignTo: str
    confidence: float

@app.post("/api/ml/classify-ticket", response_model=TicketClassification)
async def classify_ticket(data: TicketData):
    try:
        text = data.ticketText.lower()
        
        # Simple keyword-based classification
        if any(word in text for word in ['login', 'password', 'authentication', 'access']):
            category = 'authentication'
            assign_to = 'security_team'
            priority = 'high'
            confidence = 0.9
        elif any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            assign_to = 'engineering_team'
            priority = 'high'
            confidence = 0.85
        elif any(word in text for word in ['feature', 'request', 'enhancement']):
            category = 'feature_request'
            assign_to = 'product_team'
            priority = 'low'
            confidence = 0.8
        else:
            category = 'general'
            assign_to = 'support_team'
            priority = 'medium'
            confidence = 0.7
        
        return TicketClassification(
            category=category,
            priority=priority,
            assignTo=assign_to,
            confidence=confidence
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port
