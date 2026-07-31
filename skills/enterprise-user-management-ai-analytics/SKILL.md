---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, task management, and workflow automation
triggers:
  - "set up enterprise user management system"
  - "implement AI-based user analytics and risk detection"
  - "create user management dashboard with task tracking"
  - "build admin panel with role-based access control"
  - "integrate ML service for burnout and anomaly detection"
  - "configure JWT authentication for user management"
  - "deploy user management system with AI insights"
  - "implement kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with AI-powered insights. It provides role-based access control, task tracking with Kanban boards, support ticket management, and ML-driven analytics including risk prediction, anomaly detection, burnout analysis, and project delay forecasting.

**Architecture:**
- **Frontend**: React.js application
- **Backend**: Node.js REST API server
- **ML Service**: FastAPI-based AI/ML service
- **Database**: MongoDB
- **Authentication**: JWT tokens

## Installation

### Prerequisites

```bash
# Required tools
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Frontend runs at http://localhost:3000
```

## Backend API Reference

### Authentication Endpoints

```javascript
// User Registration
POST /api/auth/register
Body: {
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user" // or "admin"
}

// User Login
POST /api/auth/login
Body: {
  "email": "john@example.com",
  "password": "securePassword123"
}
Response: {
  "token": "jwt_token_here",
  "user": { ... }
}

// Get Current User
GET /api/auth/me
Headers: { "Authorization": "Bearer <token>" }
```

### User Management Endpoints

```javascript
// Get All Users (Admin only)
GET /api/users
Headers: { "Authorization": "Bearer <admin_token>" }

// Get User by ID
GET /api/users/:userId
Headers: { "Authorization": "Bearer <token>" }

// Update User
PUT /api/users/:userId
Headers: { "Authorization": "Bearer <token>" }
Body: {
  "name": "Updated Name",
  "email": "updated@example.com"
}

// Delete User (Admin only)
DELETE /api/users/:userId
Headers: { "Authorization": "Bearer <admin_token>" }
```

### Task Management Endpoints

```javascript
// Create Task
POST /api/tasks
Headers: { "Authorization": "Bearer <token>" }
Body: {
  "title": "Implement feature X",
  "description": "Complete implementation",
  "assignedTo": "userId",
  "status": "todo", // todo, in_progress, done
  "priority": "high", // low, medium, high
  "dueDate": "2026-05-01"
}

// Get Tasks
GET /api/tasks?status=in_progress&assignedTo=userId
Headers: { "Authorization": "Bearer <token>" }

// Update Task Status
PUT /api/tasks/:taskId
Headers: { "Authorization": "Bearer <token>" }
Body: {
  "status": "in_progress"
}

// Track Time
POST /api/tasks/:taskId/time
Headers: { "Authorization": "Bearer <token>" }
Body: {
  "duration": 3600 // seconds
}
```

### Support Ticket Endpoints

```javascript
// Create Ticket
POST /api/tickets
Headers: { "Authorization": "Bearer <token>" }
Body: {
  "title": "Cannot access dashboard",
  "description": "Getting 404 error",
  "priority": "medium",
  "category": "technical"
}

// Get Tickets
GET /api/tickets?status=open
Headers: { "Authorization": "Bearer <token>" }

// Update Ticket
PUT /api/tickets/:ticketId
Headers: { "Authorization": "Bearer <token>" }
Body: {
  "status": "resolved",
  "resolution": "Fixed authentication issue"
}
```

## Frontend Usage Patterns

### Authentication Component

```javascript
// src/components/Login.jsx
import React, { useState } from 'react';
import axios from 'axios';

const Login = () => {
  const [credentials, setCredentials] = useState({ email: '', password: '' });
  const [error, setError] = useState('');

  const handleLogin = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_API_URL}/auth/login`,
        credentials
      );
      
      // Store JWT token
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
      
      // Redirect based on role
      if (response.data.user.role === 'admin') {
        window.location.href = '/admin/dashboard';
      } else {
        window.location.href = '/dashboard';
      }
    } catch (err) {
      setError(err.response?.data?.message || 'Login failed');
    }
  };

  return (
    <form onSubmit={handleLogin}>
      <input
        type="email"
        placeholder="Email"
        value={credentials.email}
        onChange={(e) => setCredentials({ ...credentials, email: e.target.value })}
        required
      />
      <input
        type="password"
        placeholder="Password"
        value={credentials.password}
        onChange={(e) => setCredentials({ ...credentials, password: e.target.value })}
        required
      />
      {error && <p className="error">{error}</p>}
      <button type="submit">Login</button>
    </form>
  );
};

export default Login;
```

### API Service Helper

```javascript
// src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

// Create axios instance with default config
const api = axios.create({
  baseURL: API_URL,
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

// Handle 401 responses
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import api from '../services/api';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await api.get('/tasks');
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], in_progress: [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await api.put(`/tasks/${taskId}`, { status: newStatus });
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const renderColumn = (status, title) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      {tasks[status]?.map((task) => (
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
  );

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('in_progress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};

export default KanbanBoard;
```

### Time Tracking Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect, useRef } from 'react';
import api from '../services/api';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [seconds, setSeconds] = useState(0);
  const intervalRef = useRef(null);

  useEffect(() => {
    return () => {
      if (intervalRef.current) clearInterval(intervalRef.current);
    };
  }, []);

  const startTimer = () => {
    setIsRunning(true);
    intervalRef.current = setInterval(() => {
      setSeconds((prev) => prev + 1);
    }, 1000);
  };

  const stopTimer = async () => {
    setIsRunning(false);
    clearInterval(intervalRef.current);
    
    if (seconds > 0) {
      try {
        await api.post(`/tasks/${taskId}/time`, { duration: seconds });
        setSeconds(0);
      } catch (error) {
        console.error('Failed to save time:', error);
      }
    }
  };

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const remainingSeconds = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes
      .toString()
      .padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <button onClick={isRunning ? stopTimer : startTimer}>
        {isRunning ? 'Stop' : 'Start'}
      </button>
    </div>
  );
};

export default TimeTracker;
```

## ML Service API Reference

### Risk Prediction

```python
# ML Service endpoint for risk prediction
POST /api/ml/predict-risk
Body: {
  "userId": "user_id_here",
  "features": {
    "task_count": 15,
    "overdue_tasks": 3,
    "avg_completion_time": 48,
    "ticket_count": 5,
    "last_activity_hours": 72
  }
}
Response: {
  "risk_level": "medium",
  "risk_score": 0.65,
  "factors": ["high_overdue_rate", "reduced_activity"]
}
```

### Anomaly Detection

```python
POST /api/ml/detect-anomaly
Body: {
  "userId": "user_id_here",
  "behavior": {
    "login_time": "03:00",
    "login_location": "unusual_ip",
    "failed_attempts": 5,
    "unusual_actions": ["bulk_delete", "permission_change"]
  }
}
Response: {
  "is_anomaly": true,
  "anomaly_score": 0.89,
  "alert_level": "high"
}
```

### Burnout Detection

```python
POST /api/ml/detect-burnout
Body: {
  "userId": "user_id_here",
  "metrics": {
    "weekly_hours": 65,
    "tasks_completed": 42,
    "stress_indicators": ["missed_deadlines", "late_responses"],
    "workload_trend": "increasing"
  }
}
Response: {
  "burnout_risk": "high",
  "burnout_score": 0.78,
  "recommendations": [
    "Reduce task load",
    "Schedule time off",
    "Redistribute urgent tasks"
  ]
}
```

### ML Service Implementation Example

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from typing import Dict, List
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Load models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class RiskPredictionRequest(BaseModel):
    userId: str
    features: Dict[str, float]

class RiskPredictionResponse(BaseModel):
    risk_level: str
    risk_score: float
    factors: List[str]

@app.post("/api/ml/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Extract features
        features = [
            request.features.get("task_count", 0),
            request.features.get("overdue_tasks", 0),
            request.features.get("avg_completion_time", 0),
            request.features.get("ticket_count", 0),
            request.features.get("last_activity_hours", 0)
        ]
        
        # Load risk prediction model
        model = joblib.load(f"{MODEL_PATH}/risk_model.pkl")
        
        # Predict
        risk_score = model.predict_proba([features])[0][1]
        
        # Determine risk level
        if risk_score < 0.3:
            risk_level = "low"
        elif risk_score < 0.7:
            risk_level = "medium"
        else:
            risk_level = "high"
        
        # Identify risk factors
        factors = []
        if request.features.get("overdue_tasks", 0) > 2:
            factors.append("high_overdue_rate")
        if request.features.get("last_activity_hours", 0) > 48:
            factors.append("reduced_activity")
        if request.features.get("task_count", 0) > 20:
            factors.append("high_workload")
        
        return RiskPredictionResponse(
            risk_level=risk_level,
            risk_score=float(risk_score),
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class BurnoutRequest(BaseModel):
    userId: str
    metrics: Dict[str, any]

class BurnoutResponse(BaseModel):
    burnout_risk: str
    burnout_score: float
    recommendations: List[str]

@app.post("/api/ml/detect-burnout", response_model=BurnoutResponse)
async def detect_burnout(request: BurnoutRequest):
    try:
        weekly_hours = request.metrics.get("weekly_hours", 40)
        tasks_completed = request.metrics.get("tasks_completed", 0)
        stress_indicators = len(request.metrics.get("stress_indicators", []))
        
        # Simple burnout scoring (replace with trained model)
        burnout_score = 0.0
        
        if weekly_hours > 60:
            burnout_score += 0.4
        elif weekly_hours > 50:
            burnout_score += 0.2
        
        if tasks_completed > 40:
            burnout_score += 0.3
        
        burnout_score += stress_indicators * 0.1
        burnout_score = min(burnout_score, 1.0)
        
        # Determine risk level
        if burnout_score < 0.4:
            burnout_risk = "low"
            recommendations = ["Maintain current pace"]
        elif burnout_score < 0.7:
            burnout_risk = "medium"
            recommendations = ["Monitor workload", "Take regular breaks"]
        else:
            burnout_risk = "high"
            recommendations = [
                "Reduce task load immediately",
                "Schedule time off",
                "Redistribute urgent tasks",
                "Consult with manager"
            ]
        
        return BurnoutResponse(
            burnout_risk=burnout_risk,
            burnout_score=float(burnout_score),
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Backend Implementation Patterns

### MongoDB Schema Models

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please provide a name'],
    trim: true
  },
  email: {
    type: String,
    required: [true, 'Please provide an email'],
    unique: true,
    lowercase: true,
    match: [/^\S+@\S+\.\S+$/, 'Please provide a valid email']
  },
  password: {
    type: String,
    required: [true, 'Please provide a password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  position: String,
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

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, 'Task title is required'],
    trim: true
  },
  description: {
    type: String,
    required: [true, 'Task description is required']
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedBy: {
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
  dueDate: Date,
  timeTracked: {
    type: Number,
    default: 0 // in seconds
  },
  tags: [String],
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
});

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['open', 'in_progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  category: {
    type: String,
    enum: ['technical', 'access', 'feature_request', 'bug', 'other'],
    required: true
  },
  aiClassification: {
    category: String,
    confidence: Number
  },
  resolution: String,
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  try {
    let token;
    
    // Check for token in headers
    if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
      token = req.headers.authorization.split(' ')[1];
    }
    
    if (!token) {
      return res.status(401).json({
        success: false,
        message: 'Not authorized to access this route'
      });
    }
    
    // Verify token
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    // Get user from token
    req.user = await User.findById(decoded.id);
    
    if (!req.user) {
      return res.status(401).json({
        success: false,
        message: 'User not found'
      });
    }
    
    next();
  } catch (error) {
    return res.status(401).json({
      success: false,
      message: 'Not authorized to access this route'
    });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        message: `User role ${req.user.role} is not authorized to access this route`
      });
    }
    next();
  };
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      assignedBy: req.user._id
    });
    
    res.status(201).json({
      success: true,
      data: task
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: error.message
    });
  }
};

exports.getTasks = async (req, res) => {
  try {
    const query = {};
    
    // Filter by role
    if (req.user.role !== 'admin') {
      query.assignedTo = req.user._id;
    }
    
    // Apply filters from query params
    if (req.query.status) query.status = req.query.status;
    if (req.query.priority) query.priority = req.query.priority;
    if (req.query.assignedTo) query.assignedTo = req.query.assignedTo;
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('assignedBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.status(200).json({
      success: true,
      count: tasks.length,
      data: tasks
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: error.message
    });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({
        success: false,
        message: 'Task not found'
      });
    }
    
    // Check authorization
    if (req.user.role !== 'admin' && 
        task.assignedTo.toString() !== req.user._id.toString()) {
      return res.status(403).json({
        success: false,
        message: 'Not authorized to update this task'
      });
    }
    
    const updatedTask = await Task.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    );
    
    res.status(200).json({
      success: true,
      data: updatedTask
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: error.message
    });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({
        success: false,
        message: 'Task not found'
      });
    }
    
    task.timeTracked += req.body.duration;
    await task.save();
    
    res.status(200).json({
      success: true,
      data: task
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: error.message
    });
  }
};
```

### Ticket Controller with AI Integration

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

exports.createTicket = async (req, res) => {
  try {
    // Create ticket
    const ticket = await Ticket.create({
      ...req.body,
      createdBy: req.user._id
    });
    
    // Get AI classification
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/ml/classify-ticket`,
        {
          title: ticket.title,
          description: ticket.description
        }
      );
      
      ticket.aiClassification = mlResponse.data;
      await ticket.save();
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    res.status(201).json({
      success: true,
      data: ticket
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: error.message
    });
  }
};

exports.getTickets = async (req, res) => {
  try {
    const query = {};
    
    // Non-admin users see only their tickets
    if (req.user.role !== 'admin') {
      query.createdBy = req.user._id;
    }
    
    if (req.query.status) query.status = req.query.status;
    if (req.query.priority) query.priority = req.query.priority;
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'name email')
      .populate('assignedTo', 'name email')
      .sort({ createdAt: -1 });
    
    res.status(200).json({
      success: true,
      count: tickets.length,
      data: tickets
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: error.message
    });
  }
};
```

## Configuration

### Environment Variables

**Backend (.env):**
```env
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt

# JWT
JWT_SECRET=your_secure_jwt_secret_key
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# CORS
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env):**
```env
# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt

# Models
MODEL_PATH=./models

# Logging
LOG_LEVEL=INFO

# API
API_KEY=your_ml_service_api_key
```

**Frontend (.env):**
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

## Common Patterns

### Protected Route Component

```javascript
// src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const
