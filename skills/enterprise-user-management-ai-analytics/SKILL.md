---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "configure the user management dashboard with AI insights"
  - "implement risk detection and anomaly analysis"
  - "set up task tracking with burnout detection"
  - "create a user management system with ML predictions"
  - "build an admin dashboard with AI features"
  - "integrate ticket classification AI service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user management, task tracking, and support ticket systems with AI-powered insights. It provides risk detection, anomaly detection, burnout analysis, and predictive project insights through machine learning models.

**Key capabilities:**
- JWT-based authentication and role-based access control
- Kanban-style task management with time tracking
- AI-powered ticket classification and routing
- Risk prediction and anomaly detection
- Burnout detection through workload analysis
- Admin dashboard with organizational analytics

## Installation

### Prerequisites

```bash
# Required software
Node.js >= 14.x
Python >= 3.8
MongoDB >= 4.x
```

### Setup Steps

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
MODEL_PATH=./models
LOG_LEVEL=INFO
```

### Running the Application

```bash
# Terminal 1: Start MongoDB
mongod

# Terminal 2: Start Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 3: Start ML Service
cd ml-service
uvicorn main:app --reload
# Runs at http://localhost:8000

# Terminal 4: Start Frontend
cd frontend
npm start
# Runs at http://localhost:3000
```

## Core Architecture

### Backend API Structure

The backend follows RESTful API design with the following key routes:

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// User registration
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Check if user exists
    let user = await User.findOne({ email });
    if (user) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    // Create user
    user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate JWT
    const payload = {
      user: {
        id: user.id,
        role: user.role
      }
    };
    
    jwt.sign(
      payload,
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE },
      (err, token) => {
        if (err) throw err;
        res.json({ token, user: { id: user.id, username, email, role: user.role } });
      }
    );
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// User login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    // Check user exists
    let user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    // Verify password
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    // Generate JWT
    const payload = {
      user: {
        id: user.id,
        role: user.role
      }
    };
    
    jwt.sign(
      payload,
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE },
      (err, token) => {
        if (err) throw err;
        res.json({ 
          token, 
          user: { 
            id: user.id, 
            username: user.username, 
            email, 
            role: user.role 
          } 
        });
      }
    );
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

module.exports = router;
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

module.exports = function(req, res, next) {
  // Get token from header
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  // Check if no token
  if (!token) {
    return res.status(401).json({ message: 'No token, authorization denied' });
  }
  
  // Verify token
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded.user;
    next();
  } catch (err) {
    res.status(401).json({ message: 'Token is not valid' });
  }
};

// Role-based authorization
module.exports.isAdmin = function(req, res, next) {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied. Admin only.' });
  }
  next();
};
```

### Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const Task = require('../models/Task');

// Get all tasks for a user
router.get('/', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ 
      $or: [
        { assignedTo: req.user.id },
        { createdBy: req.user.id }
      ]
    }).sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// Create a task
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, priority, assignedTo, deadline, status } = req.body;
    
    const task = new Task({
      title,
      description,
      priority: priority || 'medium',
      assignedTo,
      createdBy: req.user.id,
      deadline,
      status: status || 'todo',
      timeTracking: {
        totalSeconds: 0,
        sessions: []
      }
    });
    
    await task.save();
    res.json(task);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// Update task status (Kanban movement)
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// Time tracking - start/stop timer
router.post('/:id/time', auth, async (req, res) => {
  try {
    const { action, seconds } = req.body; // action: 'start' | 'stop'
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    if (action === 'start') {
      task.timeTracking.sessions.push({
        startTime: new Date(),
        userId: req.user.id
      });
    } else if (action === 'stop' && seconds) {
      task.timeTracking.totalSeconds += seconds;
      const lastSession = task.timeTracking.sessions[task.timeTracking.sessions.length - 1];
      if (lastSession) {
        lastSession.endTime = new Date();
        lastSession.duration = seconds;
      }
    }
    
    await task.save();
    res.json(task);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

module.exports = router;
```

### Support Ticket System

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const axios = require('axios');
const Ticket = require('../models/Ticket');

// Create support ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Create ticket
    const ticket = new Ticket({
      title,
      description,
      priority: priority || 'medium',
      createdBy: req.user.id,
      status: 'open'
    });
    
    // Call ML service for ticket classification
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        {
          title,
          description
        }
      );
      
      ticket.category = mlResponse.data.category;
      ticket.suggestedAssignee = mlResponse.data.suggested_assignee;
      ticket.aiConfidence = mlResponse.data.confidence;
    } catch (mlErr) {
      console.error('ML service error:', mlErr.message);
      // Continue without AI classification
    }
    
    await ticket.save();
    res.json(ticket);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// Get all tickets (admin) or user's tickets
router.get('/', auth, async (req, res) => {
  try {
    let query = {};
    
    if (req.user.role !== 'admin') {
      query.createdBy = req.user.id;
    }
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

module.exports = router;
```

## AI/ML Service Integration

### FastAPI ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, metrics
import joblib
import os

app = FastAPI(title="Enterprise User Management AI Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")
os.makedirs(MODEL_PATH, exist_ok=True)

# Ticket classification model
try:
    ticket_classifier = joblib.load(f"{MODEL_PATH}/ticket_classifier.pkl")
except:
    ticket_classifier = None
    print("Ticket classifier not found, using rule-based fallback")

# Anomaly detector (online learning)
anomaly_detector = anomaly.HalfSpaceTrees()

class TicketInput(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    suggested_assignee: Optional[str]
    confidence: float

class UserBehaviorInput(BaseModel):
    user_id: str
    login_time: str
    login_location: str
    actions_count: int
    failed_attempts: int

class RiskPredictionInput(BaseModel):
    task_count: int
    overdue_tasks: int
    avg_completion_time: float
    ticket_count: int
    last_login_days: int

class BurnoutAnalysisInput(BaseModel):
    user_id: str
    weekly_hours: float
    tasks_completed: int
    tasks_pending: int
    stress_indicators: int

@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(ticket: TicketInput):
    """
    Classify support ticket and suggest assignee using AI
    """
    try:
        # Rule-based classification fallback
        text = f"{ticket.title} {ticket.description}".lower()
        
        category = "general"
        if any(word in text for word in ["bug", "error", "crash", "broken"]):
            category = "technical"
            suggested_assignee = "tech-team"
        elif any(word in text for word in ["account", "login", "password", "access"]):
            category = "account"
            suggested_assignee = "support-team"
        elif any(word in text for word in ["feature", "request", "enhancement"]):
            category = "feature_request"
            suggested_assignee = "product-team"
        elif any(word in text for word in ["billing", "payment", "invoice"]):
            category = "billing"
            suggested_assignee = "finance-team"
        else:
            suggested_assignee = "general-support"
        
        confidence = 0.75  # Rule-based confidence
        
        return TicketClassificationResponse(
            category=category,
            suggested_assignee=suggested_assignee,
            confidence=confidence
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(behavior: UserBehaviorInput):
    """
    Detect anomalous user behavior for security
    """
    try:
        # Create feature vector
        features = {
            'actions_count': behavior.actions_count,
            'failed_attempts': behavior.failed_attempts,
            'hour': int(behavior.login_time.split(':')[0]) if ':' in behavior.login_time else 12
        }
        
        # Score anomaly (higher = more anomalous)
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.7
        risk_level = "high" if score > 0.8 else "medium" if score > 0.5 else "low"
        
        return {
            "user_id": behavior.user_id,
            "is_anomaly": is_anomaly,
            "anomaly_score": float(score),
            "risk_level": risk_level,
            "recommended_action": "block" if is_anomaly else "monitor"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(data: RiskPredictionInput):
    """
    Predict user risk score based on activity patterns
    """
    try:
        # Calculate risk score
        risk_score = 0.0
        
        # High task count
        if data.task_count > 20:
            risk_score += 0.2
        
        # Overdue tasks ratio
        if data.task_count > 0:
            overdue_ratio = data.overdue_tasks / data.task_count
            risk_score += overdue_ratio * 0.3
        
        # Slow completion time
        if data.avg_completion_time > 5.0:  # days
            risk_score += 0.2
        
        # High ticket count
        if data.ticket_count > 10:
            risk_score += 0.15
        
        # Inactivity
        if data.last_login_days > 7:
            risk_score += 0.15
        
        risk_score = min(risk_score, 1.0)
        
        risk_level = "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        
        return {
            "risk_score": float(risk_score),
            "risk_level": risk_level,
            "factors": {
                "high_task_count": data.task_count > 20,
                "overdue_tasks": data.overdue_tasks > 0,
                "slow_completion": data.avg_completion_time > 5.0,
                "inactive": data.last_login_days > 7
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(data: BurnoutAnalysisInput):
    """
    Analyze employee burnout risk based on workload
    """
    try:
        burnout_score = 0.0
        
        # Excessive hours
        if data.weekly_hours > 50:
            burnout_score += 0.3
        elif data.weekly_hours > 45:
            burnout_score += 0.2
        
        # Task overload
        if data.tasks_pending > 15:
            burnout_score += 0.25
        
        # Low completion rate
        total_tasks = data.tasks_completed + data.tasks_pending
        if total_tasks > 0:
            completion_rate = data.tasks_completed / total_tasks
            if completion_rate < 0.5:
                burnout_score += 0.2
        
        # Stress indicators
        burnout_score += min(data.stress_indicators * 0.1, 0.3)
        
        burnout_score = min(burnout_score, 1.0)
        
        risk_level = "critical" if burnout_score > 0.8 else "high" if burnout_score > 0.6 else "moderate" if burnout_score > 0.4 else "low"
        
        recommendations = []
        if data.weekly_hours > 50:
            recommendations.append("Reduce working hours")
        if data.tasks_pending > 15:
            recommendations.append("Redistribute workload")
        if burnout_score > 0.6:
            recommendations.append("Schedule wellness check-in")
        
        return {
            "user_id": data.user_id,
            "burnout_score": float(burnout_score),
            "risk_level": risk_level,
            "recommendations": recommendations,
            "metrics": {
                "weekly_hours": data.weekly_hours,
                "task_completion_rate": data.tasks_completed / max(total_tasks, 1)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "Enterprise AI Analytics"}
```

## Frontend Integration

### React API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Create axios instance
const api = axios.create({
  baseURL: API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Add token to requests
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Auth API
export const authAPI = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (userData) => api.post('/auth/register', userData),
  getProfile: () => api.get('/auth/profile'),
};

// Tasks API
export const tasksAPI = {
  getAll: () => api.get('/tasks'),
  create: (taskData) => api.post('/tasks', taskData),
  update: (id, taskData) => api.put(`/tasks/${id}`, taskData),
  updateStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  delete: (id) => api.delete(`/tasks/${id}`),
  trackTime: (id, action, seconds) => 
    api.post(`/tasks/${id}/time`, { action, seconds }),
};

// Tickets API
export const ticketsAPI = {
  getAll: () => api.get('/tickets'),
  create: (ticketData) => api.post('/tickets', ticketData),
  update: (id, ticketData) => api.put(`/tickets/${id}`, ticketData),
  delete: (id) => api.delete(`/tickets/${id}`),
};

// Users API (Admin)
export const usersAPI = {
  getAll: () => api.get('/users'),
  create: (userData) => api.post('/users', userData),
  update: (id, userData) => api.put(`/users/${id}`, userData),
  delete: (id) => api.delete(`/users/${id}`),
};

// ML API
export const mlAPI = {
  classifyTicket: (ticketData) => 
    axios.post(`${ML_API_URL}/classify-ticket`, ticketData),
  detectAnomaly: (behaviorData) => 
    axios.post(`${ML_API_URL}/detect-anomaly`, behaviorData),
  predictRisk: (riskData) => 
    axios.post(`${ML_API_URL}/predict-risk`, riskData),
  analyzeBurnout: (burnoutData) => 
    axios.post(`${ML_API_URL}/analyze-burnout`, burnoutData),
};

export default api;
```

### Task Kanban Component

```javascript
// frontend/src/components/TaskKanban.jsx
import React, { useState, useEffect } from 'react';
import { tasksAPI } from '../services/api';
import './TaskKanban.css';

const TaskKanban = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await tasksAPI.getAll();
      const tasksByStatus = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(tasksByStatus);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const handleDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');

    if (currentStatus !== newStatus) {
      try {
        await tasksAPI.updateStatus(taskId, newStatus);
        fetchTasks(); // Refresh tasks
      } catch (error) {
        console.error('Error updating task status:', error);
      }
    }
  };

  const TaskCard = ({ task, status }) => (
    <div
      className="task-card"
      draggable
      onDragStart={(e) => handleDragStart(e, task._id, status)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>
          {task.priority}
        </span>
        {task.deadline && (
          <span className="deadline">
            Due: {new Date(task.deadline).toLocaleDateString()}
          </span>
        )}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      <div
        className="kanban-column"
        onDragOver={handleDragOver}
        onDrop={(e) => handleDrop(e, 'todo')}
      >
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => (
          <TaskCard key={task._id} task={task} status="todo" />
        ))}
      </div>

      <div
        className="kanban-column"
        onDragOver={handleDragOver}
        onDrop={(e) => handleDrop(e, 'in_progress')}
      >
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => (
          <TaskCard key={task._id} task={task} status="in_progress" />
        ))}
      </div>

      <div
        className="kanban-column"
        onDragOver={handleDragOver}
        onDrop={(e) => handleDrop(e, 'done')}
      >
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => (
          <TaskCard key={task._id} task={task} status="done" />
        ))}
      </div>
    </div>
  );
};

export default TaskKanban;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { mlAPI, tasksAPI, usersAPI } from '../services/api';
import './AIAnalytics.css';

const AIAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutAnalysis: null,
    anomalies: []
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch user data
      const tasksResponse = await tasksAPI.getAll();
      const tasks = tasksResponse.data;

      // Calculate metrics
      const overdueTasks = tasks.filter(t => 
        new Date(t.deadline) < new Date() && t.status !== 'done'
      ).length;

      const completedTasks = tasks.filter(t => t.status === 'done');
      const avgCompletionTime = completedTasks.length > 0
        ? completedTasks.reduce((sum, t) => {
            const created = new Date(t.createdAt);
            const completed = new Date(t.completedAt);
            return sum + (completed - created) / (1000 * 60 * 60 * 24);
          }, 0) / completedTasks.length
        : 0;

      // Get risk prediction
      const riskResponse = await mlAPI.predictRisk({
        task_count: tasks.length,
        overdue_tasks: overdueTasks,
        avg_completion_time: avgCompletionTime,
        ticket_count: 5, // Fetch from tickets API
        last_login_days: 1
      });

      // Get burnout analysis
      const totalHours = tasks.reduce((sum, t) => 
        sum + (t.timeTracking?.totalSeconds || 0) / 3600, 0
      );
      const weeklyHours = totalHours; // Simplified

      const burnoutResponse = await mlAPI.analyzeBurnout({
        user_id: 'current_user',
        weekly_hours: weeklyHours,
        tasks_completed: completedTasks.length,
        tasks_pending: tasks.filter(t => t.status !== 'done').length,
        stress_indicators: overdueTasks > 5 ? 3 : 1
      });

      setAnalytics({
        riskScore: riskResponse.data,
        burnoutAnalysis: burnoutResponse.data,
        anomalies: []
      });
      setLoading(false);
    } catch (error) {
      console.error('Error fetching analytics:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading AI analytics...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>

      <div className="analytics-grid">
        {/* Risk Score Card */}
        <div className="analytics-card">
          <h3>Risk Assessment</h3>
          {analytics.riskScore && (
            <>
              <div className={`risk-score ${analytics.riskScore.risk_level}`}>
                {(analytics.riskScore.risk_score * 100).toFixed(0)}%
              
