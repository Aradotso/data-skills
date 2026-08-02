---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, and task management
triggers:
  - "how do I set up the enterprise user management system"
  - "integrate AI analytics for user behavior"
  - "implement risk prediction and anomaly detection"
  - "create user management dashboard with task tracking"
  - "build ticket classification system with AI"
  - "add burnout detection to user management"
  - "setup kanban board with time tracking"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack web application for managing users, tasks, and support tickets with integrated AI capabilities including risk detection, anomaly detection, burnout analysis, and predictive project insights. Built with React, Node.js/Express, MongoDB, and FastAPI ML service.

## What This Project Does

This system provides:
- User authentication and role-based access control (Admin/User)
- Task management with Kanban boards and time tracking
- Support ticket system with AI-powered classification
- AI analytics for risk prediction, anomaly detection, and burnout analysis
- Admin dashboard for user management and organizational analytics
- Real-time notifications and audit logging

## Installation

### Prerequisites

```bash
# Required
- Node.js 14+
- Python 3.8+
- MongoDB instance
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

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
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

Create `.env` file in ml-service directory:

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

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Architecture Overview

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   React      │─────▶│  Node.js/    │─────▶│   MongoDB    │
│   Frontend   │◀─────│  Express API │◀─────│   Database   │
└──────────────┘      └──────────────┘      └──────────────┘
                             │
                             ▼
                      ┌──────────────┐
                      │   FastAPI    │
                      │  ML Service  │
                      └──────────────┘
```

## Key API Endpoints

### Authentication

```javascript
// User Login
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}

// Response
{
  "token": "jwt_token_here",
  "user": {
    "id": "user_id",
    "name": "User Name",
    "role": "user"
  }
}

// User Registration
POST /api/auth/register
{
  "name": "New User",
  "email": "newuser@example.com",
  "password": "password123",
  "role": "user"
}
```

### User Management (Admin)

```javascript
// Get All Users
GET /api/users
Headers: { Authorization: "Bearer <token>" }

// Create User
POST /api/users
{
  "name": "John Doe",
  "email": "john@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update User
PUT /api/users/:id
{
  "name": "John Updated",
  "department": "Marketing"
}

// Delete User
DELETE /api/users/:id
```

### Task Management

```javascript
// Get User Tasks
GET /api/tasks/user/:userId

// Create Task
POST /api/tasks
{
  "title": "Complete Project Documentation",
  "description": "Write comprehensive docs",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update Task Status
PUT /api/tasks/:id
{
  "status": "in-progress",
  "timeSpent": 120
}
```

### Support Tickets

```javascript
// Create Ticket
POST /api/tickets
{
  "title": "Login Issue",
  "description": "Cannot access my account",
  "priority": "medium",
  "category": "technical"
}

// Get Tickets
GET /api/tickets/user/:userId

// Admin: Get All Tickets
GET /api/tickets
```

## ML Service API

### Risk Prediction

```python
# POST /api/ml/predict-risk
# Request body
{
  "userId": "user_id",
  "loginAttempts": 5,
  "taskCompletionRate": 0.4,
  "avgTaskDelay": 3.5,
  "ticketsRaised": 10
}

# Response
{
  "riskScore": 0.75,
  "riskLevel": "high",
  "factors": ["low_completion_rate", "high_ticket_count"]
}
```

### Anomaly Detection

```python
# POST /api/ml/detect-anomaly
{
  "userId": "user_id",
  "loginTime": "2026-04-15T03:00:00Z",
  "loginLocation": "192.168.1.1",
  "activityPattern": "unusual"
}

# Response
{
  "isAnomaly": true,
  "anomalyScore": 0.85,
  "reason": "login_time_unusual"
}
```

### Burnout Detection

```python
# POST /api/ml/detect-burnout
{
  "userId": "user_id",
  "weeklyHours": 65,
  "taskCount": 25,
  "overdueCount": 8,
  "ticketFrequency": 5
}

# Response
{
  "burnoutRisk": 0.82,
  "level": "high",
  "recommendations": [
    "Reduce task load",
    "Suggest time off"
  ]
}
```

### Ticket Classification

```python
# POST /api/ml/classify-ticket
{
  "title": "Cannot reset password",
  "description": "Getting error when trying to reset my password"
}

# Response
{
  "category": "technical",
  "priority": "high",
  "suggestedAssignee": "support_team_id",
  "confidence": 0.92
}
```

## Frontend Integration Examples

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  const login = async (email, password) => {
    try {
      const response = await axios.post(`${API_URL}/api/auth/login`, {
        email,
        password
      });
      
      const { token, user } = response.data;
      localStorage.setItem('token', token);
      setToken(token);
      setUser(user);
      
      return { success: true, user };
    } catch (error) {
      return { 
        success: false, 
        error: error.response?.data?.message || 'Login failed' 
      };
    }
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return { user, token, login, logout };
};
```

### API Service Configuration

```javascript
// src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

// Configure axios instance
const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add auth token to requests
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

// User Management
export const userService = {
  getAll: () => api.get('/api/users'),
  getById: (id) => api.get(`/api/users/${id}`),
  create: (userData) => api.post('/api/users', userData),
  update: (id, userData) => api.put(`/api/users/${id}`, userData),
  delete: (id) => api.delete(`/api/users/${id}`)
};

// Task Management
export const taskService = {
  getUserTasks: (userId) => api.get(`/api/tasks/user/${userId}`),
  create: (taskData) => api.post('/api/tasks', taskData),
  update: (id, taskData) => api.put(`/api/tasks/${id}`, taskData),
  updateStatus: (id, status) => api.patch(`/api/tasks/${id}/status`, { status }),
  delete: (id) => api.delete(`/api/tasks/${id}`)
};

// Ticket Management
export const ticketService = {
  getAll: () => api.get('/api/tickets'),
  getUserTickets: (userId) => api.get(`/api/tickets/user/${userId}`),
  create: (ticketData) => api.post('/api/tickets', ticketData),
  update: (id, ticketData) => api.put(`/api/tickets/${id}`, ticketData)
};

// ML Analytics
const mlApi = axios.create({
  baseURL: ML_API_URL
});

export const mlService = {
  predictRisk: (userData) => mlApi.post('/api/ml/predict-risk', userData),
  detectAnomaly: (activityData) => mlApi.post('/api/ml/detect-anomaly', activityData),
  detectBurnout: (userMetrics) => mlApi.post('/api/ml/detect-burnout', userMetrics),
  classifyTicket: (ticketData) => mlApi.post('/api/ml/classify-ticket', ticketData)
};

export default api;
```

### Task Management Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    loadTasks();
  }, [userId]);

  const loadTasks = async () => {
    try {
      const response = await taskService.getUserTasks(userId);
      const tasksByStatus = response.data.reduce((acc, task) => {
        const status = task.status === 'in-progress' ? 'inProgress' : task.status;
        acc[status] = [...(acc[status] || []), task];
        return acc;
      }, { todo: [], inProgress: [], done: [] });
      
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Failed to load tasks:', error);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskService.updateStatus(taskId, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="task-board">
      <Column 
        title="To Do" 
        tasks={tasks.todo} 
        onStatusChange={handleStatusChange}
        targetStatus="todo"
      />
      <Column 
        title="In Progress" 
        tasks={tasks.inProgress} 
        onStatusChange={handleStatusChange}
        targetStatus="in-progress"
      />
      <Column 
        title="Done" 
        tasks={tasks.done} 
        onStatusChange={handleStatusChange}
        targetStatus="done"
      />
    </div>
  );
};

const Column = ({ title, tasks, onStatusChange, targetStatus }) => (
  <div className="task-column">
    <h3>{title}</h3>
    <div className="task-list">
      {tasks.map(task => (
        <TaskCard 
          key={task.id} 
          task={task} 
          onStatusChange={onStatusChange}
        />
      ))}
    </div>
  </div>
);

export default TaskBoard;
```

## Backend Implementation Examples

### User Model (MongoDB/Mongoose)

```javascript
// models/User.js
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
    enum: ['admin', 'user'],
    default: 'user'
  },
  department: String,
  isActive: {
    type: Boolean,
    default: true
  },
  loginAttempts: {
    type: Number,
    default: 0
  },
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  dueDate: Date,
  timeSpent: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Authentication Controller

```javascript
// controllers/authController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

const generateToken = (userId) => {
  return jwt.sign(
    { id: userId }, 
    process.env.JWT_SECRET, 
    { expiresIn: '7d' }
  );
};

exports.register = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;

    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    // Create new user
    const user = new User({
      name,
      email,
      password,
      role: role || 'user'
    });

    await user.save();

    const token = generateToken(user._id);

    res.status(201).json({
      message: 'User created successfully',
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;

    // Find user
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    // Check password
    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      user.loginAttempts += 1;
      await user.save();
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    // Update login info
    user.lastLogin = new Date();
    user.loginAttempts = 0;
    await user.save();

    const token = generateToken(user._id);

    res.json({
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');

    if (!user || !user.isActive) {
      return res.status(401).json({ message: 'Invalid token' });
    }

    req.user = user;
    req.userId = user._id;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

exports.isAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};
```

### Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');

exports.getUserTasks = async (req, res) => {
  try {
    const { userId } = req.params;
    
    const tasks = await Task.find({ assignedTo: userId })
      .populate('createdBy', 'name email')
      .sort('-createdAt');

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.createTask = async (req, res) => {
  try {
    const taskData = {
      ...req.body,
      createdBy: req.userId
    };

    const task = new Task(taskData);
    await task.save();

    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const { id } = req.params;
    
    const task = await Task.findByIdAndUpdate(
      id,
      { ...req.body, updatedAt: new Date() },
      { new: true, runValidators: true }
    );

    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
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

# Models (load or initialize)
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class RiskPredictionInput(BaseModel):
    userId: str
    loginAttempts: int
    taskCompletionRate: float
    avgTaskDelay: float
    ticketsRaised: int

class RiskPredictionOutput(BaseModel):
    riskScore: float
    riskLevel: str
    factors: List[str]

class AnomalyDetectionInput(BaseModel):
    userId: str
    loginTime: str
    loginLocation: str
    activityPattern: str

class AnomalyDetectionOutput(BaseModel):
    isAnomaly: bool
    anomalyScore: float
    reason: str

class BurnoutDetectionInput(BaseModel):
    userId: str
    weeklyHours: float
    taskCount: int
    overdueCount: int
    ticketFrequency: int

class BurnoutDetectionOutput(BaseModel):
    burnoutRisk: float
    level: str
    recommendations: List[str]

class TicketClassificationInput(BaseModel):
    title: str
    description: str

class TicketClassificationOutput(BaseModel):
    category: str
    priority: str
    suggestedAssignee: str
    confidence: float

@app.get("/")
def read_root():
    return {"message": "Enterprise User Management ML Service", "status": "running"}

@app.post("/api/ml/predict-risk", response_model=RiskPredictionOutput)
async def predict_risk(data: RiskPredictionInput):
    try:
        # Calculate risk score based on multiple factors
        risk_factors = []
        
        # Login attempts risk
        login_risk = min(data.loginAttempts / 10.0, 1.0)
        if data.loginAttempts > 5:
            risk_factors.append("high_login_attempts")
        
        # Task completion risk
        completion_risk = 1.0 - data.taskCompletionRate
        if data.taskCompletionRate < 0.5:
            risk_factors.append("low_completion_rate")
        
        # Task delay risk
        delay_risk = min(data.avgTaskDelay / 7.0, 1.0)
        if data.avgTaskDelay > 3:
            risk_factors.append("high_task_delays")
        
        # Ticket frequency risk
        ticket_risk = min(data.ticketsRaised / 15.0, 1.0)
        if data.ticketsRaised > 8:
            risk_factors.append("high_ticket_count")
        
        # Weighted risk score
        risk_score = (
            login_risk * 0.2 +
            completion_risk * 0.3 +
            delay_risk * 0.3 +
            ticket_risk * 0.2
        )
        
        # Determine risk level
        if risk_score < 0.3:
            risk_level = "low"
        elif risk_score < 0.6:
            risk_level = "medium"
        else:
            risk_level = "high"
        
        return RiskPredictionOutput(
            riskScore=round(risk_score, 2),
            riskLevel=risk_level,
            factors=risk_factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly", response_model=AnomalyDetectionOutput)
async def detect_anomaly(data: AnomalyDetectionInput):
    try:
        from datetime import datetime
        
        # Parse login time
        login_dt = datetime.fromisoformat(data.loginTime.replace('Z', '+00:00'))
        hour = login_dt.hour
        
        # Check for unusual login times (2 AM - 6 AM is suspicious)
        time_anomaly = 2 <= hour <= 6
        
        # Check for unusual activity patterns
        pattern_anomaly = data.activityPattern == "unusual"
        
        # Calculate anomaly score
        anomaly_score = 0.0
        reasons = []
        
        if time_anomaly:
            anomaly_score += 0.4
            reasons.append("login_time_unusual")
        
        if pattern_anomaly:
            anomaly_score += 0.6
            reasons.append("activity_pattern_unusual")
        
        is_anomaly = anomaly_score > 0.5
        reason = ", ".join(reasons) if reasons else "normal_activity"
        
        return AnomalyDetectionOutput(
            isAnomaly=is_anomaly,
            anomalyScore=round(anomaly_score, 2),
            reason=reason
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-burnout", response_model=BurnoutDetectionOutput)
async def detect_burnout(data: BurnoutDetectionInput):
    try:
        recommendations = []
        
        # Hours risk (40+ is concerning, 50+ is high)
        hours_risk = min((data.weeklyHours - 40) / 20.0, 1.0) if data.weeklyHours > 40 else 0
        if data.weeklyHours > 50:
            recommendations.append("Reduce weekly hours")
        
        # Task overload risk
        task_risk = min(data.taskCount / 30.0, 1.0)
        if data.taskCount > 20:
            recommendations.append("Reduce task load")
        
        # Overdue task risk
        overdue_risk = min(data.overdueCount / 10.0, 1.0)
        if data.overdueCount > 5:
            recommendations.append("Address overdue tasks")
        
        # Ticket frequency risk
        ticket_risk = min(data.ticketFrequency / 10.0, 1.0)
        if data.ticketFrequency > 5:
            recommendations.append("Investigate support needs")
        
        # Calculate burnout risk
        burnout_risk = (
            hours_risk * 0.4 +
            task_risk * 0.25 +
            overdue_risk * 0.25 +
            ticket_risk * 0.1
        )
        
        # Determine level
        if burnout_risk < 0.3:
            level = "low"
        elif burnout_risk < 0.6:
            level = "medium"
            if not recommendations:
                recommendations.append("Monitor workload")
        else:
            level = "high"
            recommendations.append("Suggest time off")
        
        return BurnoutDetectionOutput(
            burnoutRisk=round(burnout_risk, 2),
            level=level,
            recommendations=recommendations if recommendations else ["Continue current pace"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket", response_model=TicketClassificationOutput)
async def classify_ticket(data: TicketClassificationInput):
    try:
        text = f"{data.title} {data.description}".lower()
        
        # Simple keyword-based classification
        categories = {
            "technical": ["error", "bug", "crash", "not working", "broken", "issue"],
            "access": ["login", "password", "access", "permission", "locked"],
            "feature": ["feature", "request", "need", "add", "improve"],
            "general": ["question", "help", "how to", "what is"]
        }
        
        priorities = {
            "high": ["urgent", "critical", "broken", "crash", "cannot", "not working"],
            "medium": ["issue", "problem", "error", "need"],
            "low": ["question", "request", "improve"]
        }
        
        # Classify category
        category_scores = {}
        for cat, keywords in categories.items():
            score = sum(1 for keyword in keywords if keyword in text)
            category_scores[cat] = score
        
        category = max(category_scores, key=category_scores.get) if any(category_scores.values()) else "general"
        
        # Classify priority
        priority_scores = {}
        for pri, keywords in priorities.items():
            score = sum(1 for keyword in keywords if keyword in text)
            priority_scores[pri] = score
        
        priority = max(priority_scores, key=priority_scores.get) if any(priority_scores.values()) else "medium"
        
        # Suggest assignee based on category
        assignee_map = {
            "technical": "tech_support_team",
            "access": "security_team",
            "feature": "product_team",
            "general": "general_support"
        }
        
        suggested_assignee = assignee_map.get(category, "general_support")
        
        # Calculate confidence (simple heuristic)
        max_score = max(category_scores.values()) if category_scores.values() else 0
        confidence = min(0.5 + (max_score * 0.15), 0.95)
        
        return TicketClassificationOutput(
            category=category,
            priority=priority,
            suggestedAssignee=suggested_assignee,
            confidence=round(confidence, 2)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### ML Service Requirements

```txt
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
