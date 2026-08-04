---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and automated ticket routing
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create task management with burnout detection"
  - "build support ticket system with AI routing"
  - "integrate risk prediction for users"
  - "develop user dashboard with kanban board"
  - "add anomaly detection to user management"
  - "implement JWT authentication with role-based access"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack application combining user/task management with AI-driven analytics. It provides:

- **User & Admin Dashboards**: Manage users, tasks, and support tickets
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, predictive insights
- **Task Management**: Kanban boards with time tracking
- **Smart Ticketing**: AI-based classification and routing
- **Security**: JWT authentication with role-based access control

**Architecture**: React frontend + Node.js/Express backend + FastAPI ML service + MongoDB database

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
python >= 3.8
mongodb >= 4.x
npm or yarn
```

### Complete Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env  # Configure environment variables

# ML Service setup
cd ../ml-service
pip install -r requirements.txt

# Frontend setup
cd ../frontend
npm install
```

### Environment Configuration

**backend/.env**:
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ml-service/.env**:
```bash
MODEL_PATH=./models
BACKEND_URL=http://localhost:5000
LOG_LEVEL=INFO
```

**frontend/.env**:
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the System

### Start All Services

```bash
# Terminal 1: Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 2: ML Service
cd ml-service
uvicorn main:app --reload
# Runs at http://localhost:8000

# Terminal 3: Frontend
cd frontend
npm start
# Runs at http://localhost:3000
```

### Production Build

```bash
# Frontend production build
cd frontend
npm run build

# Serve with backend
cd ../backend
NODE_ENV=production npm start
```

## Backend API Usage

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = new User({ username, email, password, role: role || 'user' });
    await user.save();
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ token, user: { id: user._id, username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user || !(await user.comparePassword(password))) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### User Management (Admin)

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { authenticate, authorize } = require('../middleware/auth');

// Get all users (Admin only)
router.get('/', authenticate, authorize(['admin']), async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update user
router.put('/:id', authenticate, authorize(['admin']), async (req, res) => {
  try {
    const { username, email, role, status } = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { username, email, role, status },
      { new: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Delete user
router.delete('/:id', authenticate, authorize(['admin']), async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authenticate } = require('../middleware/auth');

// Create task
router.post('/', authenticate, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.user.id,
      priority: priority || 'medium',
      dueDate,
      status: 'todo'
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Get user tasks
router.get('/my-tasks', authenticate, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authenticate, async (req, res) => {
  try {
    const { status } = req.body; // 'todo', 'in-progress', 'done'
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { status },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Support Tickets

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const axios = require('axios');
const { authenticate } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', authenticate, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for automatic classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
      title,
      description
    });
    
    const ticket = new Ticket({
      title,
      description,
      priority: priority || 'medium',
      category: mlResponse.data.category,
      suggestedAssignee: mlResponse.data.suggestedAssignee,
      createdBy: req.user.id,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', authenticate, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

## ML Service API Usage

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
import joblib
import numpy as np
from datetime import datetime

app = FastAPI(title="Enterprise UMS AI Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models
try:
    ticket_classifier = joblib.load('models/ticket_classifier.pkl')
    risk_predictor = joblib.load('models/risk_predictor.pkl')
    anomaly_detector = joblib.load('models/anomaly_detector.pkl')
except Exception as e:
    print(f"Warning: Could not load models: {e}")

class TicketData(BaseModel):
    title: str
    description: str

class UserActivityData(BaseModel):
    userId: str
    loginHour: int
    tasksCompleted: int
    ticketsRaised: int
    avgResponseTime: float
    failedLogins: int

class BurnoutData(BaseModel):
    userId: str
    tasksAssigned: int
    hoursWorked: float
    weekendWork: int
    missedDeadlines: int

@app.get("/")
def root():
    return {"message": "Enterprise UMS AI Service", "status": "running"}

@app.post("/classify-ticket")
def classify_ticket(data: TicketData):
    """AI-based ticket classification and routing"""
    try:
        # Feature extraction from ticket text
        text = f"{data.title} {data.description}".lower()
        
        # Simple rule-based classification (replace with trained model)
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            assignee = 'tech-team'
        elif any(word in text for word in ['account', 'password', 'login', 'access']):
            category = 'account'
            assignee = 'support-team'
        elif any(word in text for word in ['payment', 'billing', 'invoice']):
            category = 'billing'
            assignee = 'finance-team'
        else:
            category = 'general'
            assignee = 'support-team'
        
        return {
            "category": category,
            "suggestedAssignee": assignee,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
def predict_risk(data: UserActivityData):
    """Predict security risk based on user behavior"""
    try:
        # Calculate risk score
        risk_score = 0
        
        # Unusual login hours (night time)
        if data.loginHour < 6 or data.loginHour > 22:
            risk_score += 20
        
        # High number of failed logins
        if data.failedLogins > 3:
            risk_score += 30
        
        # Unusual activity patterns
        if data.tasksCompleted > 50:  # Unusually high
            risk_score += 15
        
        if data.avgResponseTime < 1.0:  # Too fast, might be automated
            risk_score += 25
        
        risk_level = 'high' if risk_score > 50 else 'medium' if risk_score > 25 else 'low'
        
        return {
            "riskScore": risk_score,
            "riskLevel": risk_level,
            "factors": {
                "suspiciousLoginHour": data.loginHour < 6 or data.loginHour > 22,
                "failedLogins": data.failedLogins,
                "unusualActivity": data.tasksCompleted > 50
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
def detect_anomaly(data: UserActivityData):
    """Detect anomalous user behavior"""
    try:
        features = np.array([[
            data.loginHour,
            data.tasksCompleted,
            data.ticketsRaised,
            data.avgResponseTime,
            data.failedLogins
        ]])
        
        # Simple threshold-based anomaly detection
        is_anomaly = (
            data.failedLogins > 5 or
            data.tasksCompleted > 100 or
            data.avgResponseTime < 0.5
        )
        
        return {
            "isAnomaly": is_anomaly,
            "anomalyScore": 0.75 if is_anomaly else 0.2,
            "explanation": "Unusual activity pattern detected" if is_anomaly else "Normal activity"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
def detect_burnout(data: BurnoutData):
    """Detect employee burnout risk"""
    try:
        burnout_score = 0
        
        # High workload
        if data.tasksAssigned > 20:
            burnout_score += 30
        
        # Long hours
        if data.hoursWorked > 50:
            burnout_score += 25
        
        # Weekend work
        if data.weekendWork > 2:
            burnout_score += 20
        
        # Missed deadlines
        if data.missedDeadlines > 3:
            burnout_score += 25
        
        risk_level = 'high' if burnout_score > 60 else 'medium' if burnout_score > 30 else 'low'
        
        return {
            "burnoutScore": burnout_score,
            "riskLevel": risk_level,
            "recommendations": [
                "Reduce task assignment" if data.tasksAssigned > 20 else None,
                "Limit overtime hours" if data.hoursWorked > 50 else None,
                "Provide deadline extension" if data.missedDeadlines > 3 else None
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Frontend Integration

### React API Client

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add auth token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authAPI = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (userData) => api.post('/auth/register', userData),
  getCurrentUser: () => api.get('/auth/me')
};

export const userAPI = {
  getAllUsers: () => api.get('/users'),
  updateUser: (id, data) => api.put(`/users/${id}`, data),
  deleteUser: (id) => api.delete(`/users/${id}`)
};

export const taskAPI = {
  createTask: (taskData) => api.post('/tasks', taskData),
  getMyTasks: () => api.get('/tasks/my-tasks'),
  updateTaskStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status })
};

export const ticketAPI = {
  createTicket: (ticketData) => api.post('/tickets', ticketData),
  getMyTickets: () => api.get('/tickets/my-tickets')
};

export const mlAPI = {
  predictRisk: (activityData) => axios.post(`${ML_API_URL}/predict-risk`, activityData),
  detectBurnout: (workloadData) => axios.post(`${ML_API_URL}/detect-burnout`, workloadData),
  detectAnomaly: (activityData) => axios.post(`${ML_API_URL}/detect-anomaly`, activityData)
};

export default api;
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getMyTasks();
      const groupedTasks = {
        todo: response.data.filter(t => t.status === 'todo'),
        'in-progress': response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(groupedTasks);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskAPI.updateTaskStatus(taskId, newStatus);
      fetchTasks(); // Refresh board
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const renderColumn = (status, title) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      <div className="task-list">
        {tasks[status].map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <div className="task-actions">
              {status !== 'done' && (
                <button onClick={() => handleStatusChange(
                  task._id,
                  status === 'todo' ? 'in-progress' : 'done'
                )}>
                  Move →
                </button>
              )}
            </div>
          </div>
        ))}
      </div>
    </div>
  );

  if (loading) return <div>Loading...</div>;

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('in-progress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { mlAPI } from '../services/api';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Gather user activity data
      const activityData = {
        userId,
        loginHour: new Date().getHours(),
        tasksCompleted: 15,
        ticketsRaised: 3,
        avgResponseTime: 2.5,
        failedLogins: 0
      };

      const workloadData = {
        userId,
        tasksAssigned: 18,
        hoursWorked: 45,
        weekendWork: 1,
        missedDeadlines: 1
      };

      const [riskResult, burnoutResult, anomalyResult] = await Promise.all([
        mlAPI.predictRisk(activityData),
        mlAPI.detectBurnout(workloadData),
        mlAPI.detectAnomaly(activityData)
      ]);

      setAnalytics({
        risk: riskResult.data,
        burnout: burnoutResult.data,
        anomaly: anomalyResult.data
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
      <h2>AI-Powered Insights</h2>
      
      <div className="analytics-card">
        <h3>Security Risk Assessment</h3>
        <div className={`risk-level ${analytics.risk.riskLevel}`}>
          {analytics.risk.riskLevel.toUpperCase()}
        </div>
        <p>Risk Score: {analytics.risk.riskScore}/100</p>
      </div>

      <div className="analytics-card">
        <h3>Burnout Detection</h3>
        <div className={`burnout-level ${analytics.burnout.riskLevel}`}>
          {analytics.burnout.riskLevel.toUpperCase()}
        </div>
        <p>Burnout Score: {analytics.burnout.burnoutScore}/100</p>
        {analytics.burnout.recommendations.filter(Boolean).length > 0 && (
          <div>
            <h4>Recommendations:</h4>
            <ul>
              {analytics.burnout.recommendations.filter(Boolean).map((rec, i) => (
                <li key={i}>{rec}</li>
              ))}
            </ul>
          </div>
        )}
      </div>

      <div className="analytics-card">
        <h3>Anomaly Detection</h3>
        <p>Status: {analytics.anomaly.isAnomaly ? '⚠️ Anomaly Detected' : '✅ Normal'}</p>
        <p>{analytics.anomaly.explanation}</p>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
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
    minlength: 6
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
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
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
    default: 0 // in minutes
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

## Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id);
    
    if (!user || user.status !== 'active') {
      return res.status(401).json({ message: 'Invalid authentication' });
    }
    
    req.user = { id: user._id, role: user.role };
    next();
  } catch (error) {
    res.status(401).json({ message: 'Authentication failed' });
  }
};

exports.authorize = (roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Access denied' });
    }
    next();
  };
};
```

## Common Patterns

### Creating User with Role-Based Access

```javascript
// Admin creating a new user
const createUser = async (userData) => {
  const response = await userAPI.register({
    username: userData.username,
    email: userData.email,
    password: userData.password,
    role: userData.role || 'user' // 'user', 'admin', 'manager'
  });
  return response.data;
};
```

### Task Assignment Workflow

```javascript
// Manager assigning task to user
const assignTask = async (taskData) => {
  const task = await taskAPI.createTask({
    title: taskData.title,
    description: taskData.description,
    assignedTo: taskData.userId,
    priority: taskData.priority,
    dueDate: taskData.dueDate
  });
  return task.data;
};
```

### AI-Enhanced Ticket Creation

```javascript
// User creates ticket, AI automatically classifies it
const createSmartTicket = async (ticketData) => {
  const ticket = await ticketAPI.createTicket({
    title: ticketData.title,
    description: ticketData.description,
    priority: ticketData.priority
    // AI service automatically adds: category, suggestedAssignee
  });
  return ticket.data;
};
```

## Troubleshooting

### ML Service Connection Issues

```bash
# Check if ML service is running
curl http://localhost:8000

# Test ML endpoint
curl -X POST http://localhost:8000/classify-ticket \
  -H "Content-Type: application/json" \
  -d '{"title":"Login issue","description":"Cannot access my account"}'
```

### JWT Token Errors

```javascript
// Clear expired token
localStorage.removeItem('token');

// Verify token format
const token = localStorage.getItem('token');
console.log('Token:', token ? 'Present' : 'Missing');
```

### MongoDB Connection Issues

```bash
# Check MongoDB status
mongosh --eval "db.adminCommand('ping')"

# Verify connection string in .env
echo $MONGODB_URI
```

### CORS Issues

```javascript
// backend/server.js - Ensure CORS is configured
const cors = require('cors');
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Model Loading Errors (ML Service)

```python
# ml-service/main.py - Handle missing models gracefully
import os

MODEL_PATH = os.
