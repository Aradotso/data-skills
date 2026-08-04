---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, anomaly detection, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "create user management app with AI analytics"
  - "implement role-based access control with task tracking"
  - "build admin dashboard with user analytics"
  - "add AI-powered ticket classification"
  - "integrate ML-based anomaly detection for users"
  - "create kanban board with time tracking"
  - "implement burnout detection for team members"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack JavaScript/Node.js application that provides comprehensive user management, task tracking, and support ticket handling with integrated AI analytics. The system uses FastAPI-based ML services for intelligent insights including risk prediction, anomaly detection, burnout analysis, and predictive project analytics.

**Key Components:**
- React.js frontend with admin and user dashboards
- Node.js/Express backend with JWT authentication
- FastAPI ML service with scikit-learn and River
- MongoDB database
- REST API architecture

## Installation

### Prerequisites

```bash
# Ensure you have installed:
node -v  # v14.x or higher
npm -v   # v6.x or higher
python --version  # Python 3.8+
mongod --version  # MongoDB 4.4+
```

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

**Backend (.env):**
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend (.env):**
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

**ML Service (.env):**
```bash
# ml-service/.env
DATABASE_URL=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the Application

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start

# Terminal 2 - Frontend
cd frontend
npm start

# Terminal 3 - ML Service
cd ml-service
uvicorn main:app --reload
```

**Service URLs:**
- Frontend: http://localhost:3000
- Backend API: http://localhost:5000
- ML Service: http://localhost:8000

## Backend API Reference

### Authentication Endpoints

```javascript
// User Registration
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "SecurePass123",
  "role": "user" // or "admin"
}

// User Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "SecurePass123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Create user
POST /api/users
Headers: { "Authorization": "Bearer <token>" }
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Headers: { "Authorization": "Bearer <token>" }
{
  "name": "Jane Smith Updated",
  "role": "admin"
}

// Delete user
DELETE /api/users/:userId
Headers: { "Authorization": "Bearer <token>" }
```

### Task Management

```javascript
// Get tasks for logged-in user
GET /api/tasks
Headers: { "Authorization": "Bearer <token>" }

// Create task
POST /api/tasks
Headers: { "Authorization": "Bearer <token>" }
{
  "title": "Implement feature X",
  "description": "Add new authentication flow",
  "assignedTo": "userId",
  "priority": "high", // low, medium, high
  "status": "todo", // todo, in_progress, done
  "dueDate": "2026-05-01T00:00:00Z"
}

// Update task status
PATCH /api/tasks/:taskId
Headers: { "Authorization": "Bearer <token>" }
{
  "status": "in_progress"
}

// Log time for task
POST /api/tasks/:taskId/time
Headers: { "Authorization": "Bearer <token>" }
{
  "duration": 3600 // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { "Authorization": "Bearer <token>" }
{
  "subject": "Login issue",
  "description": "Unable to login with correct credentials",
  "priority": "high",
  "category": "technical" // technical, billing, general
}

// Get all tickets (admin)
GET /api/tickets
Headers: { "Authorization": "Bearer <token>" }

// Update ticket
PATCH /api/tickets/:ticketId
Headers: { "Authorization": "Bearer <token>" }
{
  "status": "in_progress",
  "assignedTo": "adminUserId"
}
```

## ML Service API

### AI-Powered Analytics Endpoints

```python
# Risk Prediction
POST http://localhost:8000/predict/risk
{
  "userId": "user123",
  "taskCompletionRate": 0.65,
  "avgResponseTime": 4800,
  "ticketCount": 12,
  "lateSubmissions": 3
}
# Returns: { "riskScore": 0.72, "riskLevel": "high", "factors": [...] }

# Anomaly Detection
POST http://localhost:8000/detect/anomaly
{
  "userId": "user123",
  "loginTime": "2026-04-15T03:30:00Z",
  "ipAddress": "192.168.1.100",
  "activityPattern": [...]
}
# Returns: { "isAnomaly": true, "anomalyScore": 0.85 }

# Burnout Detection
POST http://localhost:8000/detect/burnout
{
  "userId": "user123",
  "weeklyHours": 65,
  "taskLoad": 25,
  "avgTaskDuration": 8.5,
  "overtimeDays": 4
}
# Returns: { "burnoutRisk": 0.78, "level": "high", "recommendations": [...] }

# Ticket Classification
POST http://localhost:8000/classify/ticket
{
  "subject": "Cannot access dashboard",
  "description": "Getting 403 error when trying to login"
}
# Returns: { "category": "technical", "priority": "high", "suggestedAssignee": "techSupport" }
```

## Frontend Components

### Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(response.data.user);
    } catch (error) {
      localStorage.removeItem('token');
    }
    setLoading(false);
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return { user, loading, login, logout };
};
```

### Task Board Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks`);
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], in_progress: [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="task-board">
      {['todo', 'in_progress', 'done'].map(status => (
        <div key={status} className="task-column">
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select
                value={status}
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

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      const [riskRes, burnoutRes] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_API_URL}/predict/risk`, {
          userId,
          taskCompletionRate: 0.85,
          avgResponseTime: 3600,
          ticketCount: 5,
          lateSubmissions: 1
        }),
        axios.post(`${process.env.REACT_APP_ML_API_URL}/detect/burnout`, {
          userId,
          weeklyHours: 45,
          taskLoad: 12,
          avgTaskDuration: 5.2,
          overtimeDays: 1
        })
      ]);

      setAnalytics({
        risk: riskRes.data,
        burnout: burnoutRes.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
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
        <div className={`burnout-level ${analytics.burnout.level}`}>
          {analytics.burnout.level.toUpperCase()}
        </div>
        <p>Risk: {(analytics.burnout.burnoutRisk * 100).toFixed(1)}%</p>
        <ul>
          {analytics.burnout.recommendations?.map((rec, idx) => (
            <li key={idx}>{rec}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation Patterns

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }

    req.user = user;
    req.token = token;
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

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true
  },
  password: {
    type: String,
    required: true,
    minlength: 8
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

userSchema.methods.comparePassword = async function(password) {
  return await bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
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
        { assignedTo: req.user._id },
        { createdBy: req.user._id }
      ]
    }).populate('assignedTo', 'name email');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.taskId,
      req.body,
      { new: true, runValidators: true }
    );
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.logTime = async (req, res) => {
  try {
    const task = await Task.findById(req.params.taskId);
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    task.timeEntries.push({
      user: req.user._id,
      duration: req.body.duration,
      date: new Date()
    });
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

## ML Service Implementation

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime

app = FastAPI(title="Enterprise User Management ML Service")

# Load pre-trained models
risk_model = None
anomaly_detector = None

class RiskPredictionInput(BaseModel):
    userId: str
    taskCompletionRate: float
    avgResponseTime: int
    ticketCount: int
    lateSubmissions: int

class BurnoutDetectionInput(BaseModel):
    userId: str
    weeklyHours: float
    taskLoad: int
    avgTaskDuration: float
    overtimeDays: int

@app.post("/predict/risk")
async def predict_risk(data: RiskPredictionInput):
    try:
        features = np.array([[
            data.taskCompletionRate,
            data.avgResponseTime,
            data.ticketCount,
            data.lateSubmissions
        ]])
        
        # Simple risk calculation (replace with trained model)
        risk_score = (
            (1 - data.taskCompletionRate) * 0.4 +
            (data.lateSubmissions / 10) * 0.3 +
            (data.ticketCount / 20) * 0.2 +
            (min(data.avgResponseTime / 10000, 1)) * 0.1
        )
        
        risk_level = "low" if risk_score < 0.3 else "medium" if risk_score < 0.7 else "high"
        
        return {
            "riskScore": round(risk_score, 2),
            "riskLevel": risk_level,
            "factors": [
                {"name": "Task Completion", "impact": 1 - data.taskCompletionRate},
                {"name": "Late Submissions", "impact": data.lateSubmissions / 10}
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect/burnout")
async def detect_burnout(data: BurnoutDetectionInput):
    try:
        # Burnout risk calculation
        burnout_risk = (
            (max(data.weeklyHours - 40, 0) / 40) * 0.35 +
            (data.taskLoad / 30) * 0.25 +
            (data.avgTaskDuration / 10) * 0.2 +
            (data.overtimeDays / 5) * 0.2
        )
        
        level = "low" if burnout_risk < 0.3 else "medium" if burnout_risk < 0.6 else "high"
        
        recommendations = []
        if data.weeklyHours > 50:
            recommendations.append("Reduce weekly working hours")
        if data.taskLoad > 20:
            recommendations.append("Delegate some tasks")
        if data.overtimeDays > 2:
            recommendations.append("Limit overtime days")
        
        return {
            "burnoutRisk": round(burnout_risk, 2),
            "level": level,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/classify/ticket")
async def classify_ticket(subject: str, description: str):
    try:
        # Simple keyword-based classification (replace with NLP model)
        technical_keywords = ['login', 'error', 'bug', 'crash', 'access', '403', '404', '500']
        billing_keywords = ['payment', 'invoice', 'subscription', 'charge', 'refund']
        
        text = (subject + " " + description).lower()
        
        if any(keyword in text for keyword in technical_keywords):
            category = "technical"
            priority = "high"
        elif any(keyword in text for keyword in billing_keywords):
            category = "billing"
            priority = "medium"
        else:
            category = "general"
            priority = "low"
        
        return {
            "category": category,
            "priority": priority,
            "suggestedAssignee": "techSupport" if category == "technical" else "generalSupport"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Common Patterns

### Protected Routes

```javascript
// frontend/src/App.js
import React from 'react';
import { BrowserRouter, Route, Routes, Navigate } from 'react-router-dom';
import { useAuth } from './hooks/useAuth';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useAuth();
  
  if (loading) return <div>Loading...</div>;
  
  if (!user) return <Navigate to="/login" />;
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={
          <ProtectedRoute>
            <UserDashboard />
          </ProtectedRoute>
        } />
        <Route path="/admin" element={
          <ProtectedRoute adminOnly>
            <AdminDashboard />
          </ProtectedRoute>
        } />
      </Routes>
    </BrowserRouter>
  );
}
```

### API Service Layer

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_BASE = process.env.REACT_APP_API_URL;
const ML_API_BASE = process.env.REACT_APP_ML_API_URL;

// Set up axios interceptor for auth
axios.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const userAPI = {
  getAll: () => axios.get(`${API_BASE}/users`),
  create: (data) => axios.post(`${API_BASE}/users`, data),
  update: (id, data) => axios.put(`${API_BASE}/users/${id}`, data),
  delete: (id) => axios.delete(`${API_BASE}/users/${id}`)
};

export const taskAPI = {
  getAll: () => axios.get(`${API_BASE}/tasks`),
  create: (data) => axios.post(`${API_BASE}/tasks`, data),
  update: (id, data) => axios.patch(`${API_BASE}/tasks/${id}`, data),
  logTime: (id, duration) => axios.post(`${API_BASE}/tasks/${id}/time`, { duration })
};

export const mlAPI = {
  predictRisk: (data) => axios.post(`${ML_API_BASE}/predict/risk`, data),
  detectBurnout: (data) => axios.post(`${ML_API_BASE}/detect/burnout`, data),
  classifyTicket: (subject, description) => 
    axios.post(`${ML_API_BASE}/classify/ticket`, { subject, description })
};
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check if MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection in backend
# backend/config/database.js
const mongoose = require('mongoose');

mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
})
.then(() => console.log('MongoDB connected'))
.catch(err => console.error('MongoDB connection error:', err));
```

### JWT Token Expiration

```javascript
// frontend/src/services/api.js
axios.interceptors.response.use(
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

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');
const express = require('express');

const app = express();

app.use(cors({
  origin: process.env.NODE_ENV === 'production' 
    ? 'https://yourdomain.com' 
    : 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Not Responding

```bash
# Check if service is running
curl http://localhost:8000/docs

# Restart with verbose logging
cd ml-service
uvicorn main:app --reload --log-level debug

# Check Python dependencies
pip list | grep -E 'fastapi|uvicorn|scikit-learn'
```

### Task Status Not Updating

```javascript
// Ensure proper state management
const updateTaskStatus = async (taskId, newStatus) => {
  try {
    await axios.patch(`${API_BASE}/tasks/${taskId}`, { status: newStatus });
    // Refresh task list
    setTasks(prev => prev.map(task => 
      task._id === taskId ? { ...task, status: newStatus } : task
    ));
  } catch (error) {
    console.error('Update failed:', error);
    alert('Failed to update task');
  }
};
```

### Performance Optimization

```javascript
// Use pagination for large datasets
// backend/controllers/userController.js
exports.getUsers = async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 10;
  const skip = (page - 1) * limit;

  try {
    const users = await User.find()
      .select('-password')
      .skip(skip)
      .limit(limit);
    
    const total = await User.countDocuments();
    
    res.json({
      users,
      pagination: {
        page,
        limit,
        total,
        pages: Math.ceil(total / limit)
      }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```
