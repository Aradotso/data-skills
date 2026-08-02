---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "build user management dashboard with AI insights"
  - "implement task tracking and burnout detection"
  - "create admin panel with anomaly detection"
  - "add AI-powered ticket classification system"
  - "deploy user management system with ML service"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user/task management with machine learning capabilities. It provides:

- **User Management**: Role-based access control, authentication, and user profiles
- **Task Tracking**: Kanban boards, time tracking, and task assignment
- **Support Tickets**: AI-powered ticket classification and routing
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, and predictive insights
- **Admin Dashboard**: Organization-wide analytics and audit logging

The system consists of three main components:
1. **Frontend** (React.js) - User interfaces for admin and regular users
2. **Backend** (Node.js) - REST APIs, authentication, and business logic
3. **ML Service** (FastAPI/Python) - AI/ML models for analytics and predictions

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip (for ML service)
python --version
pip --version

# MongoDB (local or cloud instance)
```

### Clone and Setup

```bash
# Clone repository
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

## Configuration

### Backend Configuration

Create `.env` file in `backend/` directory:

```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=24h
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
```

### Frontend Configuration

Create `.env` file in `frontend/` directory:

```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

### ML Service Configuration

Create `.env` file in `ml-service/` directory:

```bash
# ml-service/.env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the Application

### Start Backend Server

```bash
cd backend
npm start
# Backend runs at http://localhost:5000
```

### Start ML Service

```bash
cd ml-service
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
```

### Start Frontend

```bash
cd frontend
npm start
# Frontend runs at http://localhost:3000
```

## Backend API Usage

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ token, user: { id: user._id, username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
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
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
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

const auth = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminAuth = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { auth, adminAuth };
```

### User Management Routes

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { auth, adminAuth } = require('../middleware/auth');

// Get all users (admin only)
router.get('/', auth, adminAuth, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user by ID
router.get('/:id', auth, async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update user
router.put('/:id', auth, adminAuth, async (req, res) => {
  try {
    const { username, email, role, status } = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { username, email, role, status },
      { new: true }
    ).select('-password');
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Delete user
router.delete('/:id', auth, adminAuth, async (req, res) => {
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
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { auth } = require('../middleware/auth');

// Get user tasks
router.get('/my-tasks', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', auth, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update time spent
router.patch('/:id/time', auth, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeSpent }, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Tickets

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { type: String, enum: ['technical', 'billing', 'general', 'urgent'], default: 'general' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: {
    category: String,
    confidence: Number,
    suggestedPriority: String
  },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const { auth } = require('../middleware/auth');
const axios = require('axios');

// Create ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
      title,
      description
    });
    
    const ticket = new Ticket({
      title,
      description,
      createdBy: req.user.id,
      aiClassification: mlResponse.data
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', auth, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
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
class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskAnalysisRequest(BaseModel):
    user_id: str
    activity_data: List[dict]

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    hours_worked: float
    missed_deadlines: int
    avg_task_complexity: float

# Load or initialize models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")
os.makedirs(MODEL_PATH, exist_ok=True)

# Ticket classifier
try:
    ticket_vectorizer = joblib.load(f"{MODEL_PATH}/ticket_vectorizer.pkl")
    ticket_classifier = joblib.load(f"{MODEL_PATH}/ticket_classifier.pkl")
except:
    ticket_vectorizer = TfidfVectorizer(max_features=1000)
    ticket_classifier = MultinomialNB()

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket using AI"""
    try:
        text = f"{request.title} {request.description}"
        
        # Simple rule-based classification (replace with trained model)
        categories = {
            'technical': ['bug', 'error', 'crash', 'not working', 'technical'],
            'billing': ['payment', 'invoice', 'subscription', 'charge', 'billing'],
            'urgent': ['urgent', 'critical', 'emergency', 'asap', 'immediately'],
            'general': []
        }
        
        text_lower = text.lower()
        category = 'general'
        confidence = 0.5
        
        for cat, keywords in categories.items():
            if any(keyword in text_lower for keyword in keywords):
                category = cat
                confidence = 0.85
                break
        
        # Determine priority
        priority = 'high' if category == 'urgent' else 'medium'
        
        return {
            "category": category,
            "confidence": confidence,
            "suggestedPriority": priority
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-risk")
async def analyze_risk(request: RiskAnalysisRequest):
    """Detect anomalous user behavior and risk"""
    try:
        # Simple anomaly detection based on activity patterns
        activity_count = len(request.activity_data)
        
        # Calculate risk score (0-100)
        risk_score = 0
        anomalies = []
        
        # Check for unusual activity volume
        if activity_count > 100:
            risk_score += 30
            anomalies.append("Unusually high activity volume")
        
        # Check for off-hours activity
        off_hours_count = sum(1 for a in request.activity_data 
                             if a.get('hour', 12) < 6 or a.get('hour', 12) > 22)
        if off_hours_count > activity_count * 0.3:
            risk_score += 25
            anomalies.append("High off-hours activity")
        
        # Check for failed login attempts
        failed_logins = sum(1 for a in request.activity_data 
                           if a.get('type') == 'failed_login')
        if failed_logins > 5:
            risk_score += 45
            anomalies.append("Multiple failed login attempts")
        
        risk_level = "high" if risk_score > 60 else "medium" if risk_score > 30 else "low"
        
        return {
            "user_id": request.user_id,
            "risk_score": min(risk_score, 100),
            "risk_level": risk_level,
            "anomalies": anomalies,
            "recommendation": "Monitor closely" if risk_level == "high" else "Normal"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Predict employee burnout risk"""
    try:
        # Calculate burnout score based on workload metrics
        burnout_score = 0
        factors = []
        
        # Hours worked
        if request.hours_worked > 50:
            burnout_score += 30
            factors.append("Excessive working hours")
        elif request.hours_worked > 40:
            burnout_score += 15
        
        # Missed deadlines
        if request.missed_deadlines > 3:
            burnout_score += 25
            factors.append("Multiple missed deadlines")
        elif request.missed_deadlines > 1:
            burnout_score += 10
        
        # Task complexity
        if request.avg_task_complexity > 8:
            burnout_score += 20
            factors.append("High task complexity")
        
        # Task completion rate
        if request.tasks_completed < 3:
            burnout_score += 15
            factors.append("Low productivity")
        
        burnout_level = "high" if burnout_score > 60 else "medium" if burnout_score > 30 else "low"
        
        recommendations = []
        if burnout_level == "high":
            recommendations = [
                "Reduce workload immediately",
                "Schedule one-on-one meeting",
                "Consider task redistribution"
            ]
        elif burnout_level == "medium":
            recommendations = [
                "Monitor workload closely",
                "Encourage work-life balance"
            ]
        
        return {
            "user_id": request.user_id,
            "burnout_score": min(burnout_score, 100),
            "burnout_level": burnout_level,
            "contributing_factors": factors,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Frontend Integration

### API Client Setup

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_URL = process.env.REACT_APP_ML_SERVICE_URL;

// Create axios instance with auth
const api = axios.create({
  baseURL: API_URL
});

// Add auth token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (userData) => api.post('/auth/register', userData),
  logout: () => localStorage.removeItem('token')
};

export const userService = {
  getAll: () => api.get('/users'),
  getById: (id) => api.get(`/users/${id}`),
  update: (id, data) => api.put(`/users/${id}`, data),
  delete: (id) => api.delete(`/users/${id}`)
};

export const taskService = {
  getMyTasks: () => api.get('/tasks/my-tasks'),
  create: (taskData) => api.post('/tasks', taskData),
  updateStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  updateTime: (id, timeSpent) => api.patch(`/tasks/${id}/time`, { timeSpent })
};

export const ticketService = {
  create: (ticketData) => api.post('/tickets', ticketData),
  getMyTickets: () => api.get('/tickets/my-tickets'),
  getAll: () => api.get('/tickets')
};

export const mlService = {
  analyzeBurnout: (data) => axios.post(`${ML_URL}/analyze-burnout`, data),
  analyzeRisk: (data) => axios.post(`${ML_URL}/analyze-risk`, data)
};

export default api;
```

### React Dashboard Component

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import { taskService, mlService } from '../services/api';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [burnoutAnalysis, setBurnoutAnalysis] = useState(null);
  const [activeTimer, setActiveTimer] = useState(null);
  const [timeElapsed, setTimeElapsed] = useState(0);

  useEffect(() => {
    loadTasks();
    loadBurnoutAnalysis();
  }, []);

  const loadTasks = async () => {
    try {
      const response = await taskService.getMyTasks();
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Failed to load tasks:', error);
    }
  };

  const loadBurnoutAnalysis = async () => {
    try {
      const userId = localStorage.getItem('userId');
      const response = await mlService.analyzeBurnout({
        user_id: userId,
        tasks_completed: 10,
        hours_worked: 45,
        missed_deadlines: 2,
        avg_task_complexity: 7
      });
      setBurnoutAnalysis(response.data);
    } catch (error) {
      console.error('Failed to load burnout analysis:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await taskService.updateStatus(taskId, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Failed to move task:', error);
    }
  };

  const startTimer = (taskId) => {
    setActiveTimer(taskId);
    const interval = setInterval(() => {
      setTimeElapsed(prev => prev + 1);
    }, 1000);
    return interval;
  };

  const stopTimer = async (taskId, intervalId) => {
    clearInterval(intervalId);
    try {
      await taskService.updateTime(taskId, Math.floor(timeElapsed / 60));
      setActiveTimer(null);
      setTimeElapsed(0);
    } catch (error) {
      console.error('Failed to update time:', error);
    }
  };

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {burnoutAnalysis && (
        <div className={`burnout-alert ${burnoutAnalysis.burnout_level}`}>
          <h3>Burnout Risk: {burnoutAnalysis.burnout_level.toUpperCase()}</h3>
          <p>Score: {burnoutAnalysis.burnout_score}/100</p>
          {burnoutAnalysis.recommendations.map((rec, i) => (
            <p key={i}>• {rec}</p>
          ))}
        </div>
      )}

      <div className="kanban-board">
        <div className="column">
          <h2>To Do ({tasks.todo.length})</h2>
          {tasks.todo.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => moveTask(task._id, 'in-progress')}>
                Start
              </button>
            </div>
          ))}
        </div>

        <div className="column">
          <h2>In Progress ({tasks.inProgress.length})</h2>
          {tasks.inProgress.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <p>Time: {task.timeSpent} min</p>
              {activeTimer === task._id ? (
                <button onClick={() => stopTimer(task._id, activeTimer)}>
                  Stop Timer ({Math.floor(timeElapsed / 60)}:{(timeElapsed % 60).toString().padStart(2, '0')})
                </button>
              ) : (
                <button onClick={() => startTimer(task._id)}>
                  Start Timer
                </button>
              )}
              <button onClick={() => moveTask(task._id, 'done')}>
                Complete
              </button>
            </div>
          ))}
        </div>

        <div className="column">
          <h2>Done ({tasks.done.length})</h2>
          {tasks.done.map(task => (
            <div key={task._id} className="task-card completed">
              <h4>{task.title}</h4>
              <p>Time spent: {task.timeSpent} min</p>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { userService, taskService, mlService } from '../services/api';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskAnalysis, setRiskAnalysis] = useState([]);
  const [stats, setStats] = useState({
    totalUsers: 0,
    activeUsers: 0,
    totalTasks: 0,
    openTickets: 0
  });

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      const usersRes = await userService.getAll();
      setUsers(usersRes.data);
      
      // Analyze risk for each user
      const riskPromises = usersRes.data.map(async (user) => {
        try {
          const riskRes = await mlService.analyzeRisk({
            user_id: user._id,
            activity_data: user.recentActivity || []
          });
          return { userId: user._id, ...riskRes.data };
        } catch {
          return null;
        }
      });
      
      const risks = await Promise.all(riskPromises);
      setRiskAnalysis(risks.filter(r => r !== null));
      
      setStats({
        totalUsers: usersRes.data.length,
        activeUsers: usersRes.data.filter(u => u.status === 'active').length,
        totalTasks: 0, // Load from tasks endpoint
        openTickets: 0 // Load from tickets endpoint
      });
    } catch (error) {
      console.error('Failed to load dashboard data:', error);
    }
  };

  const highRiskUsers = riskAnalysis.filter(r => r.risk_level === 'high');

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-value">{stats.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Active Users</h3>
          <p className="stat-value">{stats.activeUsers}</p>
        </div>
        <div className="stat-card">
          <h3>High Risk Users</h3>
          <p className="stat-value alert">{highRiskUsers.length}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p className="stat-value">{stats.openTickets}</p>
        </div>
      </div>

      {highRiskUsers.length > 0 && (
        <div className="alerts-section">
          <h2>🚨 Security Alerts</h2>
          {highRiskUsers.map(risk => {
            const user = users.find(u => u._id === risk.user_id);
            return (
              <div key={risk.user_id} className="alert-card">
                <h4>{user?.username || 'Unknown User'}</h4>
                <p>Risk Score: {risk.risk_score}/100</p>
                <ul>
                  {risk.anomalies.map((anom
