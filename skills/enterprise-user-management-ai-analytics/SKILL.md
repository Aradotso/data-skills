---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "help me build a user management dashboard"
  - "how do I integrate AI analytics into user management"
  - "set up enterprise user management with task tracking"
  - "implement AI-based ticket classification system"
  - "create a kanban board with user roles"
  - "build burnout detection for employee monitoring"
  - "add risk prediction to user management app"
  - "integrate AI insights into task management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines user administration, task management, and support ticket handling with AI-powered analytics. The system provides risk detection, anomaly detection, burnout analysis, and predictive insights using machine learning models.

**Architecture:**
- Frontend: React.js
- Backend: Node.js with REST APIs
- ML Service: FastAPI with scikit-learn and River
- Database: MongoDB
- Authentication: JWT tokens

## Installation

### Prerequisites
- Node.js 14+
- Python 3.8+
- MongoDB running locally or accessible

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
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
BACKEND_URL=http://localhost:5000
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
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Key Features & Usage

### User Authentication

**Frontend Login Component (React):**

```javascript
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const loginUser = async (email, password) => {
  try {
    const response = await axios.post(`${API_URL}/auth/login`, {
      email,
      password
    });
    
    // Store JWT token
    localStorage.setItem('token', response.data.token);
    localStorage.setItem('user', JSON.stringify(response.data.user));
    
    return response.data;
  } catch (error) {
    throw error.response.data;
  }
};

const getAuthHeader = () => {
  const token = localStorage.getItem('token');
  return {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  };
};

export { loginUser, getAuthHeader };
```

**Backend Authentication Middleware (Node.js):**

```javascript
const jwt = require('jsonwebtoken');

const authMiddleware = async (req, res, next) => {
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

### User Management API

**Backend User Controller:**

```javascript
const User = require('../models/User');
const bcrypt = require('bcryptjs');

// Create new user (Admin only)
const createUser = async (req, res) => {
  try {
    const { name, email, role, department } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }
    
    // Generate temporary password
    const tempPassword = Math.random().toString(36).slice(-8);
    const hashedPassword = await bcrypt.hash(tempPassword, 10);
    
    const user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user',
      department,
      createdBy: req.user.id
    });
    
    await user.save();
    
    res.status(201).json({
      user: user.toJSON(),
      tempPassword // Send via secure channel in production
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Get all users with filters
const getUsers = async (req, res) => {
  try {
    const { role, department, status } = req.query;
    
    const filter = {};
    if (role) filter.role = role;
    if (department) filter.department = department;
    if (status) filter.status = status;
    
    const users = await User.find(filter)
      .select('-password')
      .sort({ createdAt: -1 });
    
    res.json({ users, count: users.length });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

module.exports = { createUser, getUsers };
```

### Task Management with Kanban

**Frontend Kanban Board:**

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from './auth';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks`,
        getAuthHeader()
      );
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}`,
        { status: newStatus },
        getAuthHeader()
      );
      
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(column => (
        <div key={column} className="kanban-column">
          <h3>{column.toUpperCase()}</h3>
          {tasks[column].map(task => (
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

export default KanbanBoard;
```

**Backend Task Routes:**

```javascript
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
      priority: priority || 'medium',
      status: 'todo',
      createdBy: req.user.id,
      dueDate
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const filter = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user.id };
    
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id', authMiddleware, async (req, res) => {
  try {
    const { status, priority } = req.body;
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    if (status) task.status = status;
    if (priority) task.priority = priority;
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### AI-Powered Ticket Classification

**ML Service - Ticket Classifier (FastAPI):**

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import pickle
import os

app = FastAPI()

class TicketRequest(BaseModel):
    title: str
    description: str
    priority: str = "medium"

class TicketClassification(BaseModel):
    category: str
    assigned_department: str
    urgency_score: float
    suggested_assignee: str = None

# Load or initialize model
MODEL_PATH = os.getenv('MODEL_PATH', './models')
vectorizer = TfidfVectorizer(max_features=1000)
classifier = MultinomialNB()

# Category mapping
CATEGORY_DEPT_MAP = {
    'technical': 'IT',
    'billing': 'Finance',
    'hr': 'Human Resources',
    'access': 'IT Security',
    'general': 'Support'
}

@app.post("/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketRequest):
    try:
        # Combine title and description
        text = f"{ticket.title} {ticket.description}"
        
        # Simple keyword-based classification
        text_lower = text.lower()
        
        if any(word in text_lower for word in ['password', 'login', 'access', 'permission']):
            category = 'access'
        elif any(word in text_lower for word in ['bug', 'error', 'crash', 'not working']):
            category = 'technical'
        elif any(word in text_lower for word in ['invoice', 'payment', 'billing']):
            category = 'billing'
        elif any(word in text_lower for word in ['leave', 'salary', 'benefits']):
            category = 'hr'
        else:
            category = 'general'
        
        # Calculate urgency score
        urgency_keywords = ['urgent', 'critical', 'asap', 'emergency']
        urgency_score = sum(1 for word in urgency_keywords if word in text_lower) * 0.3
        if ticket.priority == 'high':
            urgency_score += 0.4
        
        return TicketClassification(
            category=category,
            assigned_department=CATEGORY_DEPT_MAP[category],
            urgency_score=min(urgency_score, 1.0)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

**Backend Integration with ML Service:**

```javascript
const axios = require('axios');

const classifyTicket = async (ticketData) => {
  try {
    const response = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      {
        title: ticketData.title,
        description: ticketData.description,
        priority: ticketData.priority
      }
    );
    
    return response.data;
  } catch (error) {
    console.error('ML classification error:', error.message);
    // Fallback to default classification
    return {
      category: 'general',
      assigned_department: 'Support',
      urgency_score: 0.5
    };
  }
};

// Create ticket with AI classification
router.post('/tickets', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Get AI classification
    const classification = await classifyTicket({ title, description, priority });
    
    const ticket = new Ticket({
      title,
      description,
      priority,
      category: classification.category,
      department: classification.assigned_department,
      urgencyScore: classification.urgency_score,
      createdBy: req.user.id,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Burnout Detection

**ML Service - Burnout Analysis:**

```python
from datetime import datetime, timedelta
from pydantic import BaseModel
from typing import List

class WorkloadData(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_in_progress: int
    avg_hours_per_day: float
    days_without_break: int
    ticket_volume: int

class BurnoutAnalysis(BaseModel):
    risk_level: str  # low, medium, high
    risk_score: float
    factors: List[str]
    recommendations: List[str]

@app.post("/analyze-burnout", response_model=BurnoutAnalysis)
async def analyze_burnout(data: WorkloadData):
    risk_score = 0.0
    factors = []
    recommendations = []
    
    # Hours worked analysis
    if data.avg_hours_per_day > 10:
        risk_score += 0.3
        factors.append("Excessive working hours")
        recommendations.append("Reduce daily work hours to 8-9 hours")
    
    # Task overload
    if data.tasks_in_progress > 10:
        risk_score += 0.25
        factors.append("Too many concurrent tasks")
        recommendations.append("Delegate or postpone non-critical tasks")
    
    # No breaks
    if data.days_without_break > 14:
        risk_score += 0.25
        factors.append("Extended period without breaks")
        recommendations.append("Schedule time off within next week")
    
    # Ticket volume
    if data.ticket_volume > 20:
        risk_score += 0.2
        factors.append("High support ticket volume")
        recommendations.append("Distribute tickets among team members")
    
    # Determine risk level
    if risk_score >= 0.7:
        risk_level = "high"
    elif risk_score >= 0.4:
        risk_level = "medium"
    else:
        risk_level = "low"
    
    return BurnoutAnalysis(
        risk_level=risk_level,
        risk_score=min(risk_score, 1.0),
        factors=factors if factors else ["No significant risk factors detected"],
        recommendations=recommendations if recommendations else ["Maintain current work-life balance"]
    )
```

**Frontend Burnout Dashboard:**

```javascript
import React, { useEffect, useState } from 'react';
import axios from 'axios';

const BurnoutMonitor = ({ userId }) => {
  const [analysis, setAnalysis] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchBurnoutAnalysis();
  }, [userId]);

  const fetchBurnoutAnalysis = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/analytics/burnout/${userId}`,
        getAuthHeader()
      );
      
      setAnalysis(response.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching burnout analysis:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analysis...</div>;
  if (!analysis) return <div>No data available</div>;

  const getRiskColor = (level) => {
    switch (level) {
      case 'high': return '#ff4444';
      case 'medium': return '#ffaa00';
      case 'low': return '#44ff44';
      default: return '#888888';
    }
  };

  return (
    <div className="burnout-monitor">
      <h2>Burnout Risk Analysis</h2>
      
      <div 
        className="risk-indicator"
        style={{ 
          backgroundColor: getRiskColor(analysis.risk_level),
          padding: '20px',
          borderRadius: '8px'
        }}
      >
        <h3>Risk Level: {analysis.risk_level.toUpperCase()}</h3>
        <p>Score: {(analysis.risk_score * 100).toFixed(1)}%</p>
      </div>

      <div className="risk-factors">
        <h4>Risk Factors:</h4>
        <ul>
          {analysis.factors.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
      </div>

      <div className="recommendations">
        <h4>Recommendations:</h4>
        <ul>
          {analysis.recommendations.map((rec, idx) => (
            <li key={idx}>{rec}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default BurnoutMonitor;
```

### Anomaly Detection

**ML Service - User Behavior Anomaly Detection:**

```python
from river import anomaly
from river import preprocessing
import numpy as np

class AnomalyRequest(BaseModel):
    user_id: str
    login_time: str
    ip_address: str
    actions_per_minute: float
    failed_login_attempts: int
    data_access_volume: int

class AnomalyResult(BaseModel):
    is_anomaly: bool
    anomaly_score: float
    suspicious_indicators: List[str]

# Initialize online learning anomaly detector
detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

@app.post("/detect-anomaly", response_model=AnomalyResult)
async def detect_anomaly(request: AnomalyRequest):
    try:
        # Parse login time
        login_hour = int(request.login_time.split(':')[0])
        
        suspicious_indicators = []
        
        # Check for unusual login time
        if login_hour < 6 or login_hour > 22:
            suspicious_indicators.append("Login at unusual hour")
        
        # Failed login attempts
        if request.failed_login_attempts > 3:
            suspicious_indicators.append("Multiple failed login attempts")
        
        # High action rate
        if request.actions_per_minute > 50:
            suspicious_indicators.append("Unusually high activity rate")
        
        # Excessive data access
        if request.data_access_volume > 1000:
            suspicious_indicators.append("Large volume of data accessed")
        
        # Create feature vector
        features = {
            'login_hour': login_hour,
            'actions_per_minute': request.actions_per_minute,
            'failed_attempts': request.failed_login_attempts,
            'data_volume': request.data_access_volume
        }
        
        # Get anomaly score
        score = detector.score_one(features)
        detector.learn_one(features)
        
        # Normalize score
        anomaly_score = min(max(score, 0), 1)
        
        is_anomaly = anomaly_score > 0.7 or len(suspicious_indicators) >= 2
        
        return AnomalyResult(
            is_anomaly=is_anomaly,
            anomaly_score=anomaly_score,
            suspicious_indicators=suspicious_indicators if suspicious_indicators else ["No suspicious activity detected"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Analytics Dashboard

**Backend Analytics Endpoints:**

```javascript
// Get organization-wide analytics
router.get('/analytics/dashboard', authMiddleware, adminOnly, async (req, res) => {
  try {
    const totalUsers = await User.countDocuments();
    const activeUsers = await User.countDocuments({ status: 'active' });
    
    const tasks = await Task.aggregate([
      {
        $group: {
          _id: '$status',
          count: { $sum: 1 }
        }
      }
    ]);
    
    const tickets = await Ticket.aggregate([
      {
        $group: {
          _id: '$status',
          count: { $sum: 1 }
        }
      }
    ]);
    
    // Get burnout risk users
    const highRiskUsers = await User.find({ burnoutRisk: 'high' })
      .select('name email department');
    
    res.json({
      users: {
        total: totalUsers,
        active: activeUsers,
        inactive: totalUsers - activeUsers
      },
      tasks: tasks.reduce((acc, t) => {
        acc[t._id] = t.count;
        return acc;
      }, {}),
      tickets: tickets.reduce((acc, t) => {
        acc[t._id] = t.count;
        return acc;
      }, {}),
      highRiskUsers
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

## Database Models

**User Model (MongoDB/Mongoose):**

```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
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
    enum: ['admin', 'user', 'manager'],
    default: 'user'
  },
  department: {
    type: String,
    required: true
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  burnoutRisk: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'low'
  },
  lastLogin: Date,
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  }
}, { timestamps: true });

userSchema.methods.toJSON = function() {
  const user = this.toObject();
  delete user.password;
  return user;
};

module.exports = mongoose.model('User', userSchema);
```

**Task Model:**

```javascript
const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
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
  timeTracked: {
    type: Number,
    default: 0
  }
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

**Ticket Model:**

```javascript
const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'hr', 'access', 'general'],
    required: true
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in_progress', 'resolved', 'closed'],
    default: 'open'
  },
  department: String,
  urgencyScore: Number,
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  }
}, { timestamps: true });

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### Role-Based Access Control

```javascript
// Middleware for different role levels
const requireRole = (...roles) => {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Authentication required' });
    }
    
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        error: `Access denied. Required roles: ${roles.join(', ')}` 
      });
    }
    
    next();
  };
};

// Usage in routes
router.post('/users', authMiddleware, requireRole('admin'), createUser);
router.get('/analytics', authMiddleware, requireRole('admin', 'manager'), getAnalytics);
```

### Time Tracking

```javascript
// Frontend time tracker component
const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const saveTime = async () => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`,
        { timeTracked: seconds },
        getAuthHeader()
      );
      setSeconds(0);
      setIsRunning(false);
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };

  return (
    <div>
      <p>Time: {Math.floor(seconds / 3600)}h {Math.floor((seconds % 3600) / 60)}m {seconds % 60}s</p>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={saveTime}>Save</button>
    </div>
  );
};
```

### Audit Logging

```javascript
const auditSchema = new mongoose.Schema({
  action: String,
  performedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  targetType: String,
  targetId: String,
  changes: Object,
  ipAddress: String,
  timestamp: {
    type: Date,
    default: Date.now
  }
});

const createAuditLog = async (action, userId, targetType, targetId, changes, ipAddress) => {
  await AuditLog.create({
    action,
    performedBy: userId,
    targetType,
    targetId,
    changes,
    ipAddress
  });
};

// Usage example
router.delete('/users/:id', authMiddleware, adminOnly, async (req, res) => {
  const user = await User.findById(req.params.id);
  await user.remove();
  
  await createAuditLog(
    'DELETE_USER',
    req.user.id,
    'User',
    req.params.id,
    { deleted: user.toJSON() },
    req.ip
  );
  
  res.json({ message: 'User deleted' });
});
```

## Troubleshooting

### JWT Token Issues

**Problem:** "Invalid token" or "Token expired" errors

**Solution:**
```javascript
// Implement token refresh mechanism
const refreshToken = async () => {
  try {
    const response = await axios.post(
      `${API_URL}/auth/refresh`,
      {},
      {
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('refreshToken')}`
        }
      }
    );
    
    localStorage.setItem('token', response.data.token);
    return response.data.token;
  } catch (error) {
    // Redirect to login
    window.location.href = '/login';
  }
};

// Axios interceptor for auto-retry with refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      const newToken = await refreshToken();
      error.config.headers['Authorization'] = `Bearer ${newToken}`;
      return axios.request(error.config);
    }
    return Promise.reject(error);
