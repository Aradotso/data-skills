---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics for user management
  - create a user management dashboard with AI insights
  - implement risk detection and anomaly detection for users
  - build a task management system with burnout analysis
  - set up JWT authentication for enterprise app
  - configure AI-based ticket classification system
  - deploy user management system with ML service
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack application that combines user management, task tracking, support ticketing, and AI-powered analytics. It provides:

- **User & Admin Dashboards**: Role-based access control with JWT authentication
- **Task Management**: Kanban board with time tracking and progress monitoring
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project insights
- **ML Service**: FastAPI-based machine learning microservice using scikit-learn and River

The system consists of three main components:
1. **Frontend** (React.js) - User interface
2. **Backend** (Node.js/Express) - REST API and business logic
3. **ML Service** (FastAPI) - AI/ML analytics and predictions

## Installation

### Prerequisites

```bash
# Required software
node >= 14.x
python >= 3.8
mongodb >= 4.4
```

### Full Setup

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

### Environment Configuration

**Backend (.env)**
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
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
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the System

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload
# Runs at http://localhost:8000

# Terminal 3 - Frontend
cd frontend
npm start
# Runs at http://localhost:3000
```

### Production Build

```bash
# Frontend production build
cd frontend
npm run build

# Backend production
cd backend
NODE_ENV=production node server.js

# ML Service production
cd ml-service
uvicorn main:app --host 0.0.0.0 --port 8000
```

## Backend API Reference

### Authentication

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "name": "John Doe",
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

// Response
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "507f1f77bcf86cd799439011",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user"
  }
}
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Authorization: Bearer <token>

// Create user
POST /api/users
Authorization: Bearer <token>
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:id
Authorization: Bearer <token>
{
  "name": "Jane Smith Updated",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
Authorization: Bearer <token>
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Authorization: Bearer <token>

// Create task
POST /api/tasks
Authorization: Bearer <token>
{
  "title": "Implement authentication",
  "description": "Add JWT authentication to API",
  "assignedTo": "507f1f77bcf86cd799439011",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:id/status
Authorization: Bearer <token>
{
  "status": "in_progress"
}

// Track time
POST /api/tasks/:id/time
Authorization: Bearer <token>
{
  "duration": 3600, // seconds
  "description": "Worked on authentication logic"
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer <token>
{
  "title": "Login issue",
  "description": "Cannot login with correct credentials",
  "priority": "high",
  "category": "technical"
}

// Get tickets
GET /api/tickets?status=open&priority=high
Authorization: Bearer <token>

// Update ticket
PATCH /api/tickets/:id
Authorization: Bearer <token>
{
  "status": "in_progress",
  "assignedTo": "507f1f77bcf86cd799439011"
}
```

## ML Service API Reference

### AI Analytics Endpoints

```python
# Risk prediction
POST http://localhost:8000/api/ml/predict-risk
Content-Type: application/json

{
  "user_id": "507f1f77bcf86cd799439011",
  "login_count": 150,
  "failed_logins": 5,
  "task_completion_rate": 0.85,
  "avg_response_time": 2.5,
  "last_active_hours": 12
}

# Response
{
  "risk_score": 0.23,
  "risk_level": "low",
  "factors": ["Low failed login rate", "Good completion rate"]
}
```

```python
# Anomaly detection
POST http://localhost:8000/api/ml/detect-anomaly
{
  "user_id": "507f1f77bcf86cd799439011",
  "activity_data": {
    "login_time": "03:00:00",
    "location": "Unknown",
    "device": "New Device",
    "actions_per_minute": 50
  }
}

# Response
{
  "is_anomaly": true,
  "anomaly_score": 0.87,
  "alerts": ["Unusual login time", "High activity rate"]
}
```

```python
# Burnout detection
POST http://localhost:8000/api/ml/detect-burnout
{
  "user_id": "507f1f77bcf86cd799439011",
  "workload_data": {
    "hours_worked_week": 65,
    "tasks_assigned": 25,
    "tasks_completed": 15,
    "overtime_hours": 25,
    "missed_deadlines": 3
  }
}

# Response
{
  "burnout_risk": "high",
  "burnout_score": 0.78,
  "recommendations": [
    "Reduce workload by 30%",
    "Reassign 5 tasks to other team members"
  ]
}
```

```python
# Project delay prediction
POST http://localhost:8000/api/ml/predict-delay
{
  "project_id": "proj_123",
  "tasks_total": 50,
  "tasks_completed": 20,
  "days_elapsed": 30,
  "days_remaining": 30,
  "team_size": 5,
  "avg_velocity": 0.67
}

# Response
{
  "delay_probability": 0.72,
  "estimated_delay_days": 15,
  "completion_confidence": 0.28,
  "recommendations": ["Add 2 team members", "Reduce scope"]
}
```

```python
# Ticket classification
POST http://localhost:8000/api/ml/classify-ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when trying to access admin dashboard",
  "user_role": "user"
}

# Response
{
  "category": "technical",
  "priority": "high",
  "suggested_department": "IT Support",
  "routing": "escalate"
}
```

## Frontend Integration

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

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
      const response = await axios.get(`${API_URL}/auth/me`);
      setUser(response.data);
    } catch (error) {
      console.error('Auth error:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/auth/login`, {
      email,
      password
    });
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    setUser(user);
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return { user, loading, login, logout };
};
```

### Task Management Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

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
      const response = await axios.get(`${API_URL}/tasks`);
      const tasksByStatus = {
        todo: response.data.filter(t => t.status === 'todo'),
        in_progress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const startTimer = (taskId) => {
    // Implement time tracking logic
    const startTime = Date.now();
    localStorage.setItem(`task_${taskId}_start`, startTime);
  };

  const stopTimer = async (taskId) => {
    const startTime = localStorage.getItem(`task_${taskId}_start`);
    if (startTime) {
      const duration = Math.floor((Date.now() - startTime) / 1000);
      await axios.post(`${API_URL}/tasks/${taskId}/time`, { duration });
      localStorage.removeItem(`task_${taskId}_start`);
    }
  };

  return (
    <div className="task-board">
      {Object.entries(tasks).map(([status, taskList]) => (
        <div key={status} className="task-column">
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          {taskList.map(task => (
            <div key={task.id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>
                {task.priority}
              </span>
              <button onClick={() => startTimer(task.id)}>Start</button>
              <button onClick={() => stopTimer(task.id)}>Stop</button>
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
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_URL = process.env.REACT_APP_ML_URL;
const API_URL = process.env.REACT_APP_API_URL;

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch user data
      const userResponse = await axios.get(`${API_URL}/users/${userId}/stats`);
      const userData = userResponse.data;

      // Get AI predictions
      const [riskData, burnoutData] = await Promise.all([
        axios.post(`${ML_URL}/api/ml/predict-risk`, {
          user_id: userId,
          login_count: userData.login_count,
          failed_logins: userData.failed_logins,
          task_completion_rate: userData.completion_rate,
          avg_response_time: userData.avg_response_time,
          last_active_hours: userData.last_active_hours
        }),
        axios.post(`${ML_URL}/api/ml/detect-burnout`, {
          user_id: userId,
          workload_data: {
            hours_worked_week: userData.hours_worked,
            tasks_assigned: userData.tasks_assigned,
            tasks_completed: userData.tasks_completed,
            overtime_hours: userData.overtime,
            missed_deadlines: userData.missed_deadlines
          }
        })
      ]);

      setAnalytics({
        risk: riskData.data,
        burnout: burnoutData.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="risk-section">
        <h3>Risk Assessment</h3>
        <div className={`risk-level-${analytics.risk.risk_level}`}>
          Score: {(analytics.risk.risk_score * 100).toFixed(1)}%
        </div>
        <ul>
          {analytics.risk.factors.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
      </div>

      <div className="burnout-section">
        <h3>Burnout Detection</h3>
        <div className={`burnout-${analytics.burnout.burnout_risk}`}>
          Risk: {analytics.burnout.burnout_risk}
        </div>
        <div className="recommendations">
          <h4>Recommendations:</h4>
          <ul>
            {analytics.burnout.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
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
    lowercase: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

// Method to compare password
userSchema.methods.comparePassword = async function(password) {
  return bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Get all tasks for user
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user._id })
      .populate('assignedBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      assignedBy: req.user._id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.status = req.body.status;
    if (req.body.status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Track time
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.timeEntries.push({
      user: req.user._id,
      duration: req.body.duration,
      description: req.body.description,
      timestamp: new Date()
    });

    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from river import anomaly
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models storage
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Request models
class RiskPredictionRequest(BaseModel):
    user_id: str
    login_count: int
    failed_logins: int
    task_completion_rate: float
    avg_response_time: float
    last_active_hours: int

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    activity_data: dict

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    workload_data: dict

class ProjectDelayRequest(BaseModel):
    project_id: str
    tasks_total: int
    tasks_completed: int
    days_elapsed: int
    days_remaining: int
    team_size: int
    avg_velocity: float

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    user_role: Optional[str] = "user"

# Risk prediction
@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Calculate risk score based on features
        risk_factors = []
        risk_score = 0.0
        
        # Failed login ratio
        if request.login_count > 0:
            fail_ratio = request.failed_logins / request.login_count
            if fail_ratio > 0.1:
                risk_score += 0.3
                risk_factors.append("High failed login rate")
        
        # Task completion
        if request.task_completion_rate < 0.7:
            risk_score += 0.2
            risk_factors.append("Low task completion rate")
        elif request.task_completion_rate > 0.9:
            risk_factors.append("Good completion rate")
        
        # Response time
        if request.avg_response_time > 5.0:
            risk_score += 0.15
            risk_factors.append("Slow response time")
        
        # Activity
        if request.last_active_hours > 48:
            risk_score += 0.2
            risk_factors.append("Inactive for extended period")
        elif request.last_active_hours < 24:
            risk_factors.append("Recently active")
        
        # Determine risk level
        if risk_score < 0.3:
            risk_level = "low"
        elif risk_score < 0.6:
            risk_level = "medium"
        else:
            risk_level = "high"
        
        return {
            "risk_score": round(risk_score, 2),
            "risk_level": risk_level,
            "factors": risk_factors
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Anomaly detection
@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        activity = request.activity_data
        is_anomaly = False
        anomaly_score = 0.0
        alerts = []
        
        # Check unusual login time
        login_hour = int(activity.get('login_time', '12:00:00').split(':')[0])
        if login_hour < 6 or login_hour > 22:
            is_anomaly = True
            anomaly_score += 0.3
            alerts.append("Unusual login time")
        
        # Check location
        if activity.get('location') == 'Unknown':
            is_anomaly = True
            anomaly_score += 0.2
            alerts.append("Unknown location")
        
        # Check device
        if 'New Device' in activity.get('device', ''):
            anomaly_score += 0.15
            alerts.append("New device detected")
        
        # Check activity rate
        actions_per_min = activity.get('actions_per_minute', 0)
        if actions_per_min > 30:
            is_anomaly = True
            anomaly_score += 0.35
            alerts.append("High activity rate")
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": round(anomaly_score, 2),
            "alerts": alerts
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout detection
@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        workload = request.workload_data
        burnout_score = 0.0
        recommendations = []
        
        hours_worked = workload.get('hours_worked_week', 40)
        tasks_assigned = workload.get('tasks_assigned', 10)
        tasks_completed = workload.get('tasks_completed', 8)
        overtime = workload.get('overtime_hours', 0)
        missed_deadlines = workload.get('missed_deadlines', 0)
        
        # Check work hours
        if hours_worked > 50:
            burnout_score += 0.3
            recommendations.append("Reduce workload by 30%")
        
        # Check overtime
        if overtime > 10:
            burnout_score += 0.25
            recommendations.append("Limit overtime hours")
        
        # Check task load
        completion_rate = tasks_completed / tasks_assigned if tasks_assigned > 0 else 1.0
        if completion_rate < 0.6:
            burnout_score += 0.2
            recommendations.append("Reassign 5 tasks to other team members")
        
        # Check deadlines
        if missed_deadlines > 2:
            burnout_score += 0.25
            recommendations.append("Review project timelines")
        
        # Determine risk level
        if burnout_score < 0.3:
            risk_level = "low"
        elif burnout_score < 0.6:
            risk_level = "medium"
        else:
            risk_level = "high"
        
        if not recommendations:
            recommendations.append("Continue current pace")
        
        return {
            "burnout_risk": risk_level,
            "burnout_score": round(burnout_score, 2),
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Project delay prediction
@app.post("/api/ml/predict-delay")
async def predict_delay(request: ProjectDelayRequest):
    try:
        completion_rate = request.tasks_completed / request.tasks_total
        expected_completion = request.days_elapsed / (request.days_elapsed + request.days_remaining)
        
        delay_probability = 0.0
        estimated_delay_days = 0
        recommendations = []
        
        # Compare actual vs expected
        if completion_rate < expected_completion - 0.1:
            delay_probability = 0.7
            estimated_delay_days = int((request.days_remaining * 0.3))
            recommendations.append("Add 2 team members")
        elif completion_rate < expected_completion:
            delay_probability = 0.4
            estimated_delay_days = int((request.days_remaining * 0.15))
            recommendations.append("Increase team velocity")
        
        # Check velocity
        if request.avg_velocity < 0.7:
            delay_probability += 0.2
            recommendations.append("Reduce scope")
        
        # Team size check
        if request.team_size < 3 and request.tasks_total > 30:
            delay_probability += 0.1
            recommendations.append("Expand team")
        
        delay_probability = min(delay_probability, 1.0)
        completion_confidence = 1.0 - delay_probability
        
        return {
            "delay_probability": round(delay_probability, 2),
            "estimated_delay_days": estimated_delay_days,
            "completion_confidence": round(completion_confidence, 2),
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Ticket classification
@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple keyword-based classification
        category = "general"
        priority = "medium"
        department = "Support"
        routing = "standard"
        
        # Technical keywords
        tech_keywords = ['error', '403', '500', 'crash', 'bug', 'api', 'database']
        if any(keyword in text for keyword in tech_keywords):
            category = "technical"
            priority = "high"
            department = "IT Support"
            routing = "escalate"
        
        # Access keywords
        access_keywords = ['access', 'permission', 'login', 'password', 'cannot']
        if any(
