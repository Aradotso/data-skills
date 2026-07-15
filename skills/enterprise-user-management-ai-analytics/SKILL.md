---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement user task tracking with AI insights"
  - "create admin dashboard with role-based access"
  - "add AI ticket classification and routing"
  - "build user management system with ML features"
  - "configure JWT authentication for enterprise app"
  - "implement burnout detection and risk prediction"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that provides centralized user management, task tracking, support ticketing, and AI-powered analytics. The system uses React for frontend, Node.js/Express for backend, MongoDB for data storage, and FastAPI with scikit-learn for ML services.

Key capabilities:
- JWT-based authentication and role-based access control
- Kanban-style task management with time tracking
- Support ticket system with AI classification
- AI features: risk prediction, anomaly detection, burnout analysis, predictive insights
- Admin dashboard with user management and analytics
- Real-time notifications and audit logging

## Installation

### Prerequisites

```bash
# Required installations
node --version  # v14+ required
python --version  # Python 3.8+ required
mongo --version  # MongoDB 4.4+ required
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

### Environment Configuration

Create `.env` in backend directory:

```env
# Backend Configuration
PORT=5000
NODE_ENV=development

# MongoDB
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt

# JWT Authentication
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# CORS
FRONTEND_URL=http://localhost:3000
```

Create `.env` in ml-service directory:

```env
# ML Service Configuration
API_PORT=8000
MODEL_PATH=./models
LOG_LEVEL=INFO

# Database for ML features
ML_DB_URI=mongodb://localhost:27017/ml_analytics
```

## Running the System

### Start All Services

```bash
# Terminal 1: Start MongoDB
mongod --dbpath /path/to/data/db

# Terminal 2: Start Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 3: Start ML Service
cd ml-service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Runs at http://localhost:8000

# Terminal 4: Start Frontend
cd frontend
npm start
# Runs at http://localhost:3000
```

## Backend API Reference

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "SecurePass123!",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "SecurePass123!"
}
// Returns: { token, user }

// Get current user
GET /api/auth/me
Headers: { "Authorization": "Bearer <token>" }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <admin_token>" }

// Create user
POST /api/users
Headers: { "Authorization": "Bearer <admin_token>" }
{
  "username": "jane.smith",
  "email": "jane@example.com",
  "password": "Password123!",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Headers: { "Authorization": "Bearer <admin_token>" }
{
  "role": "admin",
  "department": "Management"
}

// Delete user
DELETE /api/users/:userId
Headers: { "Authorization": "Bearer <admin_token>" }
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { "Authorization": "Bearer <token>" }

// Create task
POST /api/tasks
Headers: { "Authorization": "Bearer <token>" }
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update task status
PUT /api/tasks/:taskId
Headers: { "Authorization": "Bearer <token>" }
{
  "status": "in-progress"
}

// Track time on task
POST /api/tasks/:taskId/time
Headers: { "Authorization": "Bearer <token>" }
{
  "timeSpent": 3600  // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { "Authorization": "Bearer <token>" }
{
  "title": "Login issue",
  "description": "Cannot access dashboard after password reset",
  "priority": "medium",
  "category": "technical"
}

// Get tickets (filtered by role)
GET /api/tickets
Headers: { "Authorization": "Bearer <token>" }

// Update ticket (Admin)
PUT /api/tickets/:ticketId
Headers: { "Authorization": "Bearer <admin_token>" }
{
  "status": "resolved",
  "assignedTo": "support_user_id"
}
```

## ML Service API Reference

### AI Ticket Classification

```javascript
// Classify ticket using AI
POST http://localhost:8000/api/ml/classify-ticket
{
  "title": "Cannot access system",
  "description": "Getting 403 error when trying to login",
  "userId": "user123"
}
// Returns: { category, priority, suggestedAssignee }
```

### Risk Prediction

```javascript
// Predict user risk
POST http://localhost:8000/api/ml/predict-risk
{
  "userId": "user123",
  "metrics": {
    "loginFailures": 5,
    "taskCompletionRate": 0.45,
    "avgResponseTime": 72,  // hours
    "ticketsRaised": 12
  }
}
// Returns: { riskScore, riskLevel, factors }
```

### Anomaly Detection

```javascript
// Detect anomalous behavior
POST http://localhost:8000/api/ml/detect-anomaly
{
  "userId": "user123",
  "activity": {
    "loginTime": "2026-04-15T23:45:00Z",
    "location": "Unknown",
    "deviceId": "new_device_001"
  }
}
// Returns: { isAnomaly, anomalyScore, details }
```

### Burnout Detection

```javascript
// Analyze burnout risk
POST http://localhost:8000/api/ml/burnout-analysis
{
  "userId": "user123",
  "workload": {
    "tasksCount": 25,
    "avgHoursPerDay": 11,
    "weekendWork": 8,
    "overdueTasksCount": 10
  }
}
// Returns: { burnoutScore, riskLevel, recommendations }
```

### Predictive Insights

```javascript
// Predict project delays
POST http://localhost:8000/api/ml/predict-project
{
  "projectId": "proj123",
  "metrics": {
    "totalTasks": 50,
    "completedTasks": 15,
    "daysRemaining": 20,
    "teamSize": 5,
    "averageVelocity": 2.5
  }
}
// Returns: { delayProbability, estimatedCompletionDate, suggestions }
```

## Frontend Integration

### React Component Examples

#### Authentication Hook

```javascript
// hooks/useAuth.js
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
      const response = await axios.get('http://localhost:5000/api/auth/me');
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post('http://localhost:5000/api/auth/login', {
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

#### Task Board Component

```javascript
// components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await axios.get('http://localhost:5000/api/tasks');
    const grouped = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await axios.put(`http://localhost:5000/api/tasks/${taskId}`, {
      status: newStatus
    });
    fetchTasks();
  };

  return (
    <div className="task-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="task-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>
                {task.priority}
              </span>
              <button onClick={() => updateTaskStatus(task._id, getNextStatus(status))}>
                Move →
              </button>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

const getNextStatus = (current) => {
  const flow = { 'todo': 'in-progress', 'inProgress': 'done' };
  return flow[current];
};

export default TaskBoard;
```

#### AI Insights Dashboard

```javascript
// components/AIInsights.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIInsights = ({ userId }) => {
  const [insights, setInsights] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchInsights();
  }, [userId]);

  const fetchInsights = async () => {
    try {
      const [risk, burnout, anomaly] = await Promise.all([
        axios.post('http://localhost:8000/api/ml/predict-risk', { userId }),
        axios.post('http://localhost:8000/api/ml/burnout-analysis', { userId }),
        axios.post('http://localhost:8000/api/ml/detect-anomaly', { userId })
      ]);

      setInsights({
        risk: risk.data,
        burnout: burnout.data,
        anomaly: anomaly.data
      });
    } catch (error) {
      console.error('Failed to fetch AI insights:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading AI insights...</div>;

  return (
    <div className="ai-insights">
      <div className="insight-card">
        <h3>Risk Assessment</h3>
        <div className={`risk-level ${insights.risk.riskLevel}`}>
          {insights.risk.riskLevel.toUpperCase()}
        </div>
        <p>Score: {insights.risk.riskScore.toFixed(2)}</p>
      </div>

      <div className="insight-card">
        <h3>Burnout Analysis</h3>
        <div className={`burnout-level ${insights.burnout.riskLevel}`}>
          {insights.burnout.riskLevel.toUpperCase()}
        </div>
        <ul>
          {insights.burnout.recommendations.map((rec, idx) => (
            <li key={idx}>{rec}</li>
          ))}
        </ul>
      </div>

      <div className="insight-card">
        <h3>Anomaly Detection</h3>
        {insights.anomaly.isAnomaly ? (
          <div className="alert">⚠️ Anomalous activity detected</div>
        ) : (
          <div className="success">✓ Normal activity</div>
        )}
      </div>
    </div>
  );
};

export default AIInsights;
```

## Backend Implementation Patterns

### JWT Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      throw new Error('No authentication token');
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user) {
      throw new Error('User not found');
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        error: 'Access denied. Insufficient permissions.' 
      });
    }
    next();
  };
};

module.exports = { authenticate, authorize };
```

### Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.getTasks = async (req, res) => {
  try {
    const filter = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user._id };
    
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id,
      assignedTo: req.body.assignedTo || req.user._id
    });
    
    await task.save();
    
    // Get AI insights for the task
    try {
      const aiInsights = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/ml/analyze-task`,
        {
          taskId: task._id,
          title: task.title,
          description: task.description,
          priority: task.priority
        }
      );
      task.aiInsights = aiInsights.data;
      await task.save();
    } catch (aiError) {
      console.error('AI analysis failed:', aiError.message);
    }
    
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    // Check permissions
    if (req.user.role !== 'admin' && 
        task.assignedTo.toString() !== req.user._id.toString()) {
      return res.status(403).json({ error: 'Access denied' });
    }
    
    Object.assign(task, req.body);
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

### Ticket Controller with AI Integration

```javascript
// controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

exports.createTicket = async (req, res) => {
  try {
    const ticket = new Ticket({
      ...req.body,
      createdBy: req.user._id,
      status: 'open'
    });
    
    // AI classification
    try {
      const classification = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/ml/classify-ticket`,
        {
          title: ticket.title,
          description: ticket.description,
          userId: req.user._id
        }
      );
      
      ticket.category = classification.data.category;
      ticket.priority = classification.data.priority;
      ticket.assignedTo = classification.data.suggestedAssignee;
    } catch (aiError) {
      console.error('AI classification failed:', aiError.message);
      // Continue with default values
    }
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.getTickets = async (req, res) => {
  try {
    const filter = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user._id };
    
    const tickets = await Ticket.find(filter)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
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
from typing import Optional, Dict, List
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models (create if not exists)
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

class TicketRequest(BaseModel):
    title: str
    description: str
    userId: str

class RiskRequest(BaseModel):
    userId: str
    metrics: Dict

class BurnoutRequest(BaseModel):
    userId: str
    workload: Dict

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """AI-powered ticket classification"""
    try:
        # Feature extraction from text
        text = f"{request.title} {request.description}".lower()
        
        # Rule-based classification (can be replaced with trained model)
        category = "technical"
        if any(word in text for word in ["password", "login", "access"]):
            category = "authentication"
        elif any(word in text for word in ["bug", "error", "crash"]):
            category = "bug"
        elif any(word in text for word in ["feature", "request", "add"]):
            category = "feature_request"
        
        # Priority determination
        priority = "medium"
        if any(word in text for word in ["urgent", "critical", "asap"]):
            priority = "high"
        elif any(word in text for word in ["minor", "low"]):
            priority = "low"
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85,
            "suggestedAssignee": None
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskRequest):
    """Predict user risk based on behavior metrics"""
    try:
        metrics = request.metrics
        
        # Calculate risk score (0-1)
        risk_factors = []
        
        # Login failure rate
        login_failures = metrics.get('loginFailures', 0)
        if login_failures > 3:
            risk_factors.append(0.3)
        
        # Task completion rate
        completion_rate = metrics.get('taskCompletionRate', 1.0)
        if completion_rate < 0.5:
            risk_factors.append(0.25)
        
        # Response time
        avg_response = metrics.get('avgResponseTime', 0)
        if avg_response > 48:  # hours
            risk_factors.append(0.2)
        
        # Tickets raised
        tickets = metrics.get('ticketsRaised', 0)
        if tickets > 10:
            risk_factors.append(0.15)
        
        risk_score = sum(risk_factors)
        
        # Determine risk level
        if risk_score > 0.7:
            risk_level = "high"
        elif risk_score > 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return {
            "riskScore": risk_score,
            "riskLevel": risk_level,
            "factors": [
                {"name": "Login Failures", "impact": login_failures > 3},
                {"name": "Task Completion", "impact": completion_rate < 0.5},
                {"name": "Response Time", "impact": avg_response > 48},
                {"name": "Support Tickets", "impact": tickets > 10}
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(request: BurnoutRequest):
    """Analyze burnout risk based on workload"""
    try:
        workload = request.workload
        
        # Calculate burnout score
        score = 0
        recommendations = []
        
        # Tasks count
        tasks_count = workload.get('tasksCount', 0)
        if tasks_count > 20:
            score += 0.25
            recommendations.append("Consider redistributing tasks")
        
        # Hours per day
        hours_per_day = workload.get('avgHoursPerDay', 8)
        if hours_per_day > 10:
            score += 0.3
            recommendations.append("Reduce daily working hours")
        
        # Weekend work
        weekend_work = workload.get('weekendWork', 0)
        if weekend_work > 4:
            score += 0.25
            recommendations.append("Minimize weekend work")
        
        # Overdue tasks
        overdue = workload.get('overdueTasksCount', 0)
        if overdue > 5:
            score += 0.2
            recommendations.append("Address overdue tasks priority")
        
        # Determine risk level
        if score > 0.7:
            risk_level = "high"
        elif score > 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return {
            "burnoutScore": score,
            "riskLevel": risk_level,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: dict):
    """Detect anomalous user behavior"""
    try:
        activity = request.get('activity', {})
        
        # Simple anomaly detection based on rules
        is_anomaly = False
        anomaly_score = 0
        details = []
        
        # Check login time
        login_time = activity.get('loginTime', '')
        if login_time:
            hour = datetime.fromisoformat(login_time.replace('Z', '+00:00')).hour
            if hour < 6 or hour > 22:
                is_anomaly = True
                anomaly_score += 0.4
                details.append("Unusual login time")
        
        # Check location
        location = activity.get('location', '')
        if location == "Unknown" or "suspicious" in location.lower():
            is_anomaly = True
            anomaly_score += 0.3
            details.append("Unusual location")
        
        # Check device
        device_id = activity.get('deviceId', '')
        if 'new' in device_id.lower():
            anomaly_score += 0.3
            details.append("New device detected")
        
        return {
            "isAnomaly": is_anomaly,
            "anomalyScore": anomaly_score,
            "details": details
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Database Models

### User Model

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

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
    required: true,
    minlength: 8
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
  metadata: {
    loginCount: { type: Number, default: 0 },
    failedLoginAttempts: { type: Number, default: 0 }
  }
}, { timestamps: true });

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
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
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done', 'blocked'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
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
  dueDate: Date,
  timeTracking: {
    estimated: Number,  // minutes
    actual: Number,     // minutes
    sessions: [{
      startTime: Date,
      endTime: Date,
      duration: Number
    }]
  },
  aiInsights: {
    complexity: String,
    estimatedTime: Number,
    riskFactors: [String]
  }
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

## Common Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error.message);
    process.exit(1);
  }
};

// Handle connection errors
mongoose.connection.on('error', err => {
  console.error('MongoDB error:', err);
});

mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected');
});

module.exports = connectDB;
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### JWT Token Expiration Handling

```javascript
// frontend/utils/axios.js
import axios from 'axios';

const api = axios.create({
  baseURL: 'http://localhost:5000/api'
});

// Request interceptor
api.interceptors.request.use(
  config =>
