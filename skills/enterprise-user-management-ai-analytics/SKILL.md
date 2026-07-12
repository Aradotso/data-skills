---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - implement risk detection and anomaly detection
  - build a user management dashboard with AI
  - create task management with burnout detection
  - set up JWT authentication for enterprise app
  - deploy user management system with ML service
  - configure AI-powered ticket classification
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript/Node.js application that combines traditional user and task management with AI-powered features including risk detection, anomaly detection, burnout analysis, and predictive project insights. The system uses a microservices architecture with a React frontend, Node.js backend, and FastAPI ML service.

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+ and pip
- MongoDB instance

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
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

Create `.env` file for ML service:

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

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Architecture

The system consists of three main services:

1. **Frontend (React)**: User interface on port 3000
2. **Backend (Node.js)**: REST API on port 5000
3. **ML Service (FastAPI)**: AI analytics on port 8000

## Key Backend API Patterns

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const User = require('../models/User');

const router = express.Router();

// Register user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const hashedPassword = await bcrypt.hash(password, 10);
    
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.status(201).json({ token, user: { id: user._id, username, email, role } });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const isValid = await bcrypt.compare(password, user.password);
    if (!isValid) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }
    
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

### User Management

```javascript
// backend/routes/users.js
const express = require('express');
const User = require('../models/User');
const { authMiddleware, adminOnly } = require('../middleware/auth');

const router = express.Router();

// Get all users (admin only)
router.get('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user profile
router.get('/profile', authMiddleware, async (req, res) => {
  try {
    const user = await User.findById(req.user.userId).select('-password');
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update user (admin only)
router.put('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const { username, email, role, status } = req.body;
    
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { username, email, role, status },
      { new: true, runValidators: true }
    ).select('-password');
    
    res.json(user);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Delete user (admin only)
router.delete('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    await User.findByIdAndDelete(req.params.id);
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

const router = express.Router();

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.user.userId,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const filter = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user.userId };
    
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
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
});

// Track time
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { timeSpent } = req.body; // in minutes
    
    const task = await Task.findById(req.params.id);
    task.timeTracked = (task.timeTracked || 0) + timeSpent;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Tickets

```javascript
// backend/routes/tickets.js
const express = require('express');
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authMiddleware } = require('../middleware/auth');

const router = express.Router();

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    let category = 'general';
    let suggestedPriority = priority || 'medium';
    
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        title,
        description
      });
      
      category = mlResponse.data.category;
      suggestedPriority = mlResponse.data.priority;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      category,
      priority: suggestedPriority,
      createdBy: req.user.userId,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get tickets
router.get('/', authMiddleware, async (req, res) => {
  try {
    const filter = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.userId };
    
    const tickets = await Ticket.find(filter)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email');
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update ticket
router.patch('/:id', authMiddleware, async (req, res) => {
  try {
    const { status, assignedTo, response } = req.body;
    
    const ticket = await Ticket.findByIdAndUpdate(
      req.params.id,
      { status, assignedTo, response, updatedAt: Date.now() },
      { new: true }
    );
    
    res.json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Models storage
MODELS_PATH = os.getenv('MODEL_PATH', './models')

class TicketData(BaseModel):
    title: str
    description: str

class UserBehavior(BaseModel):
    userId: str
    loginAttempts: int
    activityPattern: List[int]
    tasksCompleted: int
    avgResponseTime: float

class WorkloadData(BaseModel):
    userId: str
    activeTasks: int
    totalHoursWorked: float
    overtimeHours: float
    missedDeadlines: int

# Ticket classification
@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """AI-based ticket classification and priority assignment"""
    try:
        # Simple keyword-based classification (in production, use trained model)
        title_lower = ticket.title.lower()
        description_lower = ticket.description.lower()
        
        # Category classification
        if any(word in title_lower or word in description_lower 
               for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            priority = 'high'
        elif any(word in title_lower or word in description_lower 
                 for word in ['access', 'permission', 'login', 'password']):
            category = 'access'
            priority = 'high'
        elif any(word in title_lower or word in description_lower 
                 for word in ['feature', 'enhancement', 'improve']):
            category = 'feature_request'
            priority = 'low'
        else:
            category = 'general'
            priority = 'medium'
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Risk detection
@app.post("/detect-risk")
async def detect_risk(behavior: UserBehavior):
    """Detect risky user behavior patterns"""
    try:
        risk_score = 0
        flags = []
        
        # Multiple failed login attempts
        if behavior.loginAttempts > 5:
            risk_score += 30
            flags.append("Multiple failed login attempts")
        
        # Unusual activity pattern
        if len(behavior.activityPattern) > 0:
            avg_activity = np.mean(behavior.activityPattern)
            if avg_activity > 100:  # Unusually high activity
                risk_score += 20
                flags.append("Unusually high activity")
        
        # Low task completion
        if behavior.tasksCompleted < 2:
            risk_score += 15
            flags.append("Low task completion rate")
        
        # Slow response times
        if behavior.avgResponseTime > 3600:  # > 1 hour
            risk_score += 10
            flags.append("Slow response times")
        
        risk_level = "low"
        if risk_score > 50:
            risk_level = "high"
        elif risk_score > 25:
            risk_level = "medium"
        
        return {
            "userId": behavior.userId,
            "riskScore": risk_score,
            "riskLevel": risk_level,
            "flags": flags,
            "recommendation": "Monitor user activity" if risk_level != "low" else "No action needed"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout detection
@app.post("/detect-burnout")
async def detect_burnout(workload: WorkloadData):
    """Detect employee burnout based on workload analysis"""
    try:
        burnout_score = 0
        indicators = []
        
        # Too many active tasks
        if workload.activeTasks > 10:
            burnout_score += 25
            indicators.append("High number of active tasks")
        
        # Excessive work hours
        if workload.totalHoursWorked > 50:
            burnout_score += 30
            indicators.append("Excessive work hours")
        
        # High overtime
        if workload.overtimeHours > 15:
            burnout_score += 25
            indicators.append("High overtime hours")
        
        # Missed deadlines
        if workload.missedDeadlines > 3:
            burnout_score += 20
            indicators.append("Multiple missed deadlines")
        
        burnout_risk = "low"
        if burnout_score > 60:
            burnout_risk = "high"
        elif burnout_score > 35:
            burnout_risk = "medium"
        
        recommendations = []
        if burnout_risk == "high":
            recommendations.append("Redistribute tasks")
            recommendations.append("Schedule 1-on-1 meeting")
            recommendations.append("Consider time off")
        elif burnout_risk == "medium":
            recommendations.append("Monitor workload closely")
            recommendations.append("Review task priorities")
        
        return {
            "userId": workload.userId,
            "burnoutScore": burnout_score,
            "burnoutRisk": burnout_risk,
            "indicators": indicators,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Anomaly detection
@app.post("/detect-anomaly")
async def detect_anomaly(behavior: UserBehavior):
    """Detect anomalous user behavior"""
    try:
        anomalies = []
        anomaly_score = 0
        
        # Check for unusual login patterns
        if behavior.loginAttempts > 10:
            anomalies.append("Excessive login attempts")
            anomaly_score += 40
        
        # Check activity patterns
        if len(behavior.activityPattern) > 0:
            pattern_std = np.std(behavior.activityPattern)
            if pattern_std > 50:
                anomalies.append("Erratic activity pattern")
                anomaly_score += 30
        
        is_anomaly = anomaly_score > 40
        
        return {
            "userId": behavior.userId,
            "isAnomaly": is_anomaly,
            "anomalyScore": anomaly_score,
            "anomalies": anomalies,
            "severity": "critical" if anomaly_score > 60 else "warning" if is_anomaly else "normal"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Frontend Integration

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_BASE = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_BASE = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_BASE,
});

const mlApi = axios.create({
  baseURL: ML_API_BASE,
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: (email, password) => 
    api.post('/auth/login', { email, password }),
  register: (userData) => 
    api.post('/auth/register', userData),
  getProfile: () => 
    api.get('/users/profile'),
};

export const userService = {
  getUsers: () => api.get('/users'),
  updateUser: (id, data) => api.put(`/users/${id}`, data),
  deleteUser: (id) => api.delete(`/users/${id}`),
};

export const taskService = {
  getTasks: () => api.get('/tasks'),
  createTask: (taskData) => api.post('/tasks', taskData),
  updateTaskStatus: (id, status) => 
    api.patch(`/tasks/${id}/status`, { status }),
  trackTime: (id, timeSpent) => 
    api.post(`/tasks/${id}/time`, { timeSpent }),
};

export const ticketService = {
  getTickets: () => api.get('/tickets'),
  createTicket: (ticketData) => api.post('/tickets', ticketData),
  updateTicket: (id, data) => api.patch(`/tickets/${id}`, data),
};

export const analyticsService = {
  detectRisk: (behaviorData) => 
    mlApi.post('/detect-risk', behaviorData),
  detectBurnout: (workloadData) => 
    mlApi.post('/detect-burnout', workloadData),
  detectAnomaly: (behaviorData) => 
    mlApi.post('/detect-anomaly', behaviorData),
  classifyTicket: (ticketData) => 
    mlApi.post('/classify-ticket', ticketData),
};

export default api;
```

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import { taskService, analyticsService } from '../services/api';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      const tasksResponse = await taskService.getTasks();
      setTasks(tasksResponse.data);

      // Calculate workload metrics
      const activeTasks = tasksResponse.data.filter(t => t.status !== 'done').length;
      const totalHours = tasksResponse.data.reduce((sum, t) => sum + (t.timeTracked || 0), 0) / 60;

      const burnoutResponse = await analyticsService.detectBurnout({
        userId: localStorage.getItem('userId'),
        activeTasks,
        totalHoursWorked: totalHours,
        overtimeHours: Math.max(0, totalHours - 40),
        missedDeadlines: tasksResponse.data.filter(t => 
          new Date(t.dueDate) < new Date() && t.status !== 'done'
        ).length
      });

      setBurnoutData(burnoutResponse.data);
      setLoading(false);
    } catch (error) {
      console.error('Failed to load dashboard:', error);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await taskService.updateTaskStatus(taskId, newStatus);
      loadDashboardData();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {burnoutData && burnoutData.burnoutRisk !== 'low' && (
        <div className={`alert ${burnoutData.burnoutRisk}`}>
          <h3>Burnout Alert: {burnoutData.burnoutRisk.toUpperCase()}</h3>
          <p>Score: {burnoutData.burnoutScore}/100</p>
          <ul>
            {burnoutData.recommendations.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>
      )}

      <div className="task-board">
        <div className="column">
          <h2>To Do</h2>
          {tasks.filter(t => t.status === 'todo').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
        
        <div className="column">
          <h2>In Progress</h2>
          {tasks.filter(t => t.status === 'inprogress').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
        
        <div className="column">
          <h2>Done</h2>
          {tasks.filter(t => t.status === 'done').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
      </div>
    </div>
  );
};

const TaskCard = ({ task, onStatusChange }) => {
  return (
    <div className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <p className="priority">{task.priority}</p>
      <select 
        value={task.status} 
        onChange={(e) => onStatusChange(task._id, e.target.value)}
      >
        <option value="todo">To Do</option>
        <option value="inprogress">In Progress</option>
        <option value="done">Done</option>
      </select>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { userService, taskService, analyticsService } from '../services/api';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskAnalysis, setRiskAnalysis] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadAnalytics();
  }, []);

  const loadAnalytics = async () => {
    try {
      const usersResponse = await userService.getUsers();
      setUsers(usersResponse.data);

      // Analyze each user for risk
      const riskPromises = usersResponse.data.map(async (user) => {
        try {
          const riskResponse = await analyticsService.detectRisk({
            userId: user._id,
            loginAttempts: user.loginAttempts || 0,
            activityPattern: user.activityPattern || [],
            tasksCompleted: user.tasksCompleted || 0,
            avgResponseTime: user.avgResponseTime || 0
          });
          return { ...user, risk: riskResponse.data };
        } catch (error) {
          return { ...user, risk: null };
        }
      });

      const analysisResults = await Promise.all(riskPromises);
      setRiskAnalysis(analysisResults);
      setLoading(false);
    } catch (error) {
      console.error('Failed to load analytics:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  const highRiskUsers = riskAnalysis.filter(u => u.risk?.riskLevel === 'high');

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p>{users.length}</p>
        </div>
        <div className="stat-card alert">
          <h3>High Risk Users</h3>
          <p>{highRiskUsers.length}</p>
        </div>
      </div>

      {highRiskUsers.length > 0 && (
        <div className="alerts-section">
          <h2>Security Alerts</h2>
          {highRiskUsers.map(user => (
            <div key={user._id} className="alert-card">
              <h4>{user.username} ({user.email})</h4>
              <p>Risk Score: {user.risk.riskScore}/100</p>
              <ul>
                {user.risk.flags.map((flag, i) => (
                  <li key={i}>{flag}</li>
                ))}
              </ul>
              <p><strong>Action:</strong> {user.risk.recommendation}</p>
            </div>
          ))}
        </div>
      )}

      <div className="user-management">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Username</th>
              <th>Email</th>
              <th>Role</th>
              <th>Risk Level</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {riskAnalysis.map(user => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td className={user.risk?.riskLevel}>
                  {user.risk?.riskLevel || 'N/A'}
                </td>
                <td>
                  <button onClick={() => handleEditUser(user._id)}>Edit</button>
                  <button onClick={() => handleDeleteUser(user._id)}>Delete</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  loginAttempts: {
    type: Number,
    default: 0
  },
  activityPattern: [Number],
  tasksCompleted: {
    type: Number,
    default: 0
  },
  avgResponseTime: {
    type: Number,
    default: 0
