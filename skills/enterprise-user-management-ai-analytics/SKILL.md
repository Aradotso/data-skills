---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management built with React, Node.js, and FastAPI
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create task management with kanban board"
  - "add risk detection and anomaly detection"
  - "build user management system with JWT authentication"
  - "integrate AI ticket classification"
  - "deploy enterprise management system"
  - "configure AI-powered admin dashboard"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack web application for managing users, tasks, and support tickets with integrated AI capabilities. It provides:

- **User Management**: Role-based access control, secure authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support System**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, alerts

**Tech Stack**: React.js (frontend), Node.js/Express (backend), FastAPI (ML service), MongoDB (database)

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
python >= 3.8
mongodb >= 4.x
npm or yarn
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

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
# Runs at http://localhost:3000
```

## Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   React     │─────▶│   Node.js   │─────▶│   MongoDB   │
│  Frontend   │      │   Backend   │      │   Database  │
└─────────────┘      └─────────────┘      └─────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │   FastAPI   │
                     │ ML Service  │
                     └─────────────┘
```

## Backend API Usage

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = await User.create({
      username,
      email,
      password, // Should be hashed with bcrypt
      role: role || 'user'
    });
    
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });
    
    res.status(201).json({ token, user: { id: user._id, username, email, role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email }).select('+password');
    if (!user || !(await user.matchPassword(password))) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });
    
    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### User Management

```javascript
// backend/routes/users.js
const express = require('express');
const User = require('../models/User');
const { protect, authorize } = require('../middleware/auth');

const router = express.Router();

// Get all users (Admin only)
router.get('/', protect, authorize('admin'), async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, count: users.length, data: users });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Get user by ID
router.get('/:id', protect, async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ success: false, message: 'User not found' });
    }
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Update user
router.put('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ success: false, message: 'User not found' });
    }
    
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Delete user
router.delete('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) {
      return res.status(404).json({ success: false, message: 'User not found' });
    }
    res.json({ success: true, data: {} });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { protect } = require('../middleware/auth');

const router = express.Router();

// Get all tasks for user
router.get('/', protect, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'username email')
      .sort('-createdAt');
    res.json({ success: true, count: tasks.length, data: tasks });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Create task
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, priority, assignedTo, dueDate } = req.body;
    
    const task = await Task.create({
      title,
      description,
      priority,
      assignedTo,
      assignedBy: req.user.id,
      dueDate,
      status: 'todo'
    });
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body; // 'todo', 'in-progress', 'done'
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ success: false, message: 'Task not found' });
    }
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
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
const { protect } = require('../middleware/auth');

const router = express.Router();

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
      title,
      description
    });
    
    const ticket = await Ticket.create({
      title,
      description,
      priority,
      createdBy: req.user.id,
      category: mlResponse.data.category,
      aiConfidence: mlResponse.data.confidence,
      status: 'open'
    });
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', protect, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .sort('-createdAt');
    res.json({ success: true, count: tickets.length, data: tickets });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

## ML Service API Usage

### FastAPI Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import joblib
import numpy as np
from datetime import datetime

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
class TicketClassifyRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_count: int
    failed_logins: int
    unusual_hours: int
    data_access_volume: float

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    average_task_duration: float
    overtime_hours: float
    tasks_overdue: int

@app.get("/")
async def root():
    return {
        "service": "Enterprise User Management ML Service",
        "version": "1.0.0",
        "endpoints": ["/classify-ticket", "/predict-risk", "/analyze-burnout", "/predict-project-delay"]
    }

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassifyRequest):
    """Classify support ticket using ML model"""
    try:
        # Simple rule-based classification (replace with actual ML model)
        text = f"{request.title} {request.description}".lower()
        
        categories = {
            "technical": ["bug", "error", "crash", "performance", "api"],
            "account": ["login", "password", "access", "permission", "profile"],
            "billing": ["payment", "invoice", "subscription", "charge"],
            "feature": ["request", "enhancement", "suggestion", "improvement"]
        }
        
        category = "general"
        confidence = 0.5
        
        for cat, keywords in categories.items():
            if any(keyword in text for keyword in keywords):
                category = cat
                confidence = 0.85
                break
        
        return {
            "category": category,
            "confidence": confidence,
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavior patterns"""
    try:
        # Risk scoring algorithm
        risk_score = 0
        
        # Failed login attempts
        if request.failed_logins > 5:
            risk_score += 30
        elif request.failed_logins > 2:
            risk_score += 15
        
        # Unusual hours activity
        if request.unusual_hours > 10:
            risk_score += 25
        elif request.unusual_hours > 5:
            risk_score += 10
        
        # Data access volume
        if request.data_access_volume > 1000:
            risk_score += 20
        elif request.data_access_volume > 500:
            risk_score += 10
        
        # Login frequency
        if request.login_count > 50:
            risk_score += 15
        
        risk_level = "low"
        if risk_score > 60:
            risk_level = "high"
        elif risk_score > 30:
            risk_level = "medium"
        
        return {
            "user_id": request.user_id,
            "risk_score": min(risk_score, 100),
            "risk_level": risk_level,
            "factors": {
                "failed_logins": request.failed_logins,
                "unusual_hours": request.unusual_hours,
                "data_access_volume": request.data_access_volume
            },
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze employee burnout risk"""
    try:
        burnout_score = 0
        
        # Overtime hours
        if request.overtime_hours > 20:
            burnout_score += 40
        elif request.overtime_hours > 10:
            burnout_score += 20
        
        # Overdue tasks
        if request.tasks_overdue > 10:
            burnout_score += 30
        elif request.tasks_overdue > 5:
            burnout_score += 15
        
        # Task completion rate
        if request.average_task_duration > 8:
            burnout_score += 20
        elif request.average_task_duration > 5:
            burnout_score += 10
        
        # High workload
        if request.tasks_completed > 100:
            burnout_score += 10
        
        burnout_level = "low"
        if burnout_score > 60:
            burnout_level = "high"
        elif burnout_score > 30:
            burnout_level = "medium"
        
        recommendations = []
        if burnout_level == "high":
            recommendations = [
                "Reduce overtime hours",
                "Redistribute workload",
                "Schedule wellness check-in",
                "Consider time off"
            ]
        elif burnout_level == "medium":
            recommendations = [
                "Monitor workload closely",
                "Check task priorities"
            ]
        
        return {
            "user_id": request.user_id,
            "burnout_score": min(burnout_score, 100),
            "burnout_level": burnout_level,
            "recommendations": recommendations,
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Frontend Implementation

### Authentication Service

```javascript
// frontend/src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

class AuthService {
  async login(email, password) {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('user', JSON.stringify(response.data));
    }
    
    return response.data;
  }
  
  async register(username, email, password, role) {
    const response = await axios.post(`${API_URL}/api/auth/register`, {
      username,
      email,
      password,
      role
    });
    
    return response.data;
  }
  
  logout() {
    localStorage.removeItem('user');
  }
  
  getCurrentUser() {
    return JSON.parse(localStorage.getItem('user'));
  }
  
  getAuthHeader() {
    const user = this.getCurrentUser();
    if (user && user.token) {
      return { Authorization: `Bearer ${user.token}` };
    }
    return {};
  }
}

export default new AuthService();
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import authService from '../services/authService';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks`, {
        headers: authService.getAuthHeader()
      });
      
      const grouped = {
        todo: response.data.data.filter(t => t.status === 'todo'),
        inProgress: response.data.data.filter(t => t.status === 'in-progress'),
        done: response.data.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const handleDrop = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: authService.getAuthHeader() }
      );
      
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div
      className="task-card"
      draggable
      onDragStart={(e) => e.dataTransfer.setData('taskId', task._id)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority priority-${task.priority}`}>
        {task.priority}
      </span>
    </div>
  );

  const Column = ({ title, status, tasks }) => (
    <div
      className="kanban-column"
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => {
        e.preventDefault();
        const taskId = e.dataTransfer.getData('taskId');
        handleDrop(taskId, status);
      }}
    >
      <h3>{title}</h3>
      <div className="task-list">
        {tasks.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <Column title="To Do" status="todo" tasks={tasks.todo} />
      <Column title="In Progress" status="in-progress" tasks={tasks.inProgress} />
      <Column title="Done" status="done" tasks={tasks.done} />
    </div>
  );
};

export default KanbanBoard;
```

### Admin Dashboard with AI Analytics

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import authService from '../services/authService';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    users: [],
    riskUsers: [],
    burnoutUsers: []
  });
  
  const API_URL = process.env.REACT_APP_API_URL;
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch users
      const usersResponse = await axios.get(`${API_URL}/api/users`, {
        headers: authService.getAuthHeader()
      });
      
      const users = usersResponse.data.data;
      
      // Analyze risk for each user
      const riskAnalysis = await Promise.all(
        users.map(async (user) => {
          try {
            const riskResponse = await axios.post(`${ML_API_URL}/predict-risk`, {
              user_id: user._id,
              login_count: user.loginCount || 0,
              failed_logins: user.failedLogins || 0,
              unusual_hours: user.unusualHours || 0,
              data_access_volume: user.dataAccessVolume || 0
            });
            
            return {
              ...user,
              riskData: riskResponse.data
            };
          } catch (error) {
            return { ...user, riskData: null };
          }
        })
      );
      
      // Filter high-risk users
      const riskUsers = riskAnalysis.filter(
        u => u.riskData && u.riskData.risk_level === 'high'
      );
      
      setAnalytics({
        users,
        riskUsers,
        burnoutUsers: [] // Similarly fetch burnout analysis
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-number">{analytics.users.length}</p>
        </div>
        
        <div className="stat-card alert">
          <h3>High Risk Users</h3>
          <p className="stat-number">{analytics.riskUsers.length}</p>
        </div>
        
        <div className="stat-card warning">
          <h3>Burnout Risk</h3>
          <p className="stat-number">{analytics.burnoutUsers.length}</p>
        </div>
      </div>
      
      <div className="risk-alerts">
        <h2>Risk Alerts</h2>
        {analytics.riskUsers.map(user => (
          <div key={user._id} className="alert-item">
            <h4>{user.username}</h4>
            <p>Risk Score: {user.riskData.risk_score}</p>
            <p>Failed Logins: {user.riskData.factors.failed_logins}</p>
            <button>View Details</button>
          </div>
        ))}
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
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  username: {
    type: String,
    required: [true, 'Please add a username'],
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: [true, 'Please add an email'],
    unique: true,
    match: [/^\S+@\S+\.\S+$/, 'Please add a valid email']
  },
  password: {
    type: String,
    required: [true, 'Please add a password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  loginCount: {
    type: Number,
    default: 0
  },
  failedLogins: {
    type: Number,
    default: 0
  },
  unusualHours: {
    type: Number,
    default: 0
  },
  dataAccessVolume: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    next();
  }
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

// Match password
UserSchema.methods.matchPassword = async function(enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, 'Please add a task title'],
    trim: true
  },
  description: {
    type: String,
    required: [true, 'Please add a description']
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
  dueDate: {
    type: Date
  },
  timeSpent: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', TaskSchema);
```

## Configuration

### Backend Environment Variables

```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_here_min_32_chars
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

### Frontend Environment Variables

```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

### ML Service Environment Variables

```bash
# ml-service/.env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Common Patterns

### Protected Route Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        message: `User role ${req.user.role} is not authorized to access this route`
      });
    }
    next();
  };
};
```

### API Client Service

```javascript
// frontend/src/services/apiClient.js
import axios from 'axios';
import authService from './authService';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Request interceptor
apiClient.interceptors.request.use(
  (config) => {
    const headers = authService.getAuthHeader();
    config.headers = { ...config.headers, ...headers };
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      authService.logout();
      window.location.href = '/login';
    }
    return Promise.
