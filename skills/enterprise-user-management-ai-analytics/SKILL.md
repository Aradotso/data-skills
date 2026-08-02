---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, ticket classification, and workload insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement JWT authentication with role-based access"
  - "create AI-powered ticket classification system"
  - "build user dashboard with task tracking"
  - "add burnout detection and risk prediction"
  - "configure ML service for user analytics"
  - "deploy user management system with AI features"
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user/task management with machine learning capabilities. It provides:

- **User Management**: JWT authentication, role-based access control (Admin/User)
- **Task Tracking**: Kanban boards, time tracking, performance monitoring
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Tech Stack**: React frontend, Node.js/Express backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites

```bash
# Required: Node.js 14+, Python 3.8+, MongoDB
node --version
python --version
mongod --version
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
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
DATABASE_URL=mongodb://localhost:27017/enterprise-ums
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   React     │────▶│  Node.js    │────▶│   MongoDB   │
│  Frontend   │     │   Backend   │     │   Database  │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   FastAPI   │
                    │  ML Service │
                    └─────────────┘
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

// Register user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = await User.create({
      name,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ token, user: { id: user._id, name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    
    if (!user || !(await bcrypt.compare(password, user.password))) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token, user: { id: user._id, name: user.name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Middleware for Protected Routes

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ message: 'Not authorized' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token invalid' });
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Not authorized for this role' });
    }
    next();
  };
};

module.exports = { protect, authorize };
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { protect, authorize } = require('../middleware/auth');

// Get user tasks
router.get('/my-tasks', protect, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task (Admin only)
router.post('/', protect, authorize('admin'), async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = await Task.create({
      title,
      description,
      assignedTo,
      assignedBy: req.user.id,
      priority,
      dueDate,
      status: 'todo'
    });
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Not authorized' });
    }
    
    task.status = status;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
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
const { protect } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { subject, description, priority } = req.body;
    
    // Call ML service for classification
    let category = 'general';
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { text: `${subject} ${description}` }
      );
      category = mlResponse.data.category;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    const ticket = await Ticket.create({
      subject,
      description,
      priority,
      category,
      createdBy: req.user.id,
      status: 'open'
    });
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get tickets
router.get('/', protect, async (req, res) => {
  try {
    const query = req.user.role === 'admin' ? {} : { createdBy: req.user.id };
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import joblib
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import os

app = FastAPI()

# Models storage
models = {}

class TicketRequest(BaseModel):
    text: str

class RiskRequest(BaseModel):
    user_id: str
    task_count: int
    overdue_tasks: int
    avg_completion_time: float
    login_frequency: float

class BurnoutRequest(BaseModel):
    user_id: str
    weekly_hours: float
    task_completion_rate: float
    missed_deadlines: int
    overtime_hours: float

@app.on_event("startup")
async def load_models():
    """Load or initialize ML models"""
    model_path = os.getenv('MODEL_PATH', './models')
    os.makedirs(model_path, exist_ok=True)
    
    # Initialize ticket classifier
    try:
        models['ticket_vectorizer'] = joblib.load(f'{model_path}/ticket_vectorizer.pkl')
        models['ticket_classifier'] = joblib.load(f'{model_path}/ticket_classifier.pkl')
    except:
        # Initialize with default model
        models['ticket_vectorizer'] = TfidfVectorizer(max_features=1000)
        models['ticket_classifier'] = MultinomialNB()

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """Classify support ticket into category"""
    try:
        vectorizer = models['ticket_vectorizer']
        classifier = models['ticket_classifier']
        
        # Transform text
        text_vector = vectorizer.transform([request.text])
        
        # Predict category
        category = classifier.predict(text_vector)[0]
        confidence = max(classifier.predict_proba(text_vector)[0])
        
        return {
            "category": category,
            "confidence": float(confidence)
        }
    except Exception as e:
        # Fallback to rule-based classification
        text_lower = request.text.lower()
        if any(word in text_lower for word in ['bug', 'error', 'crash', 'broken']):
            return {"category": "technical", "confidence": 0.7}
        elif any(word in text_lower for word in ['password', 'login', 'access']):
            return {"category": "access", "confidence": 0.7}
        elif any(word in text_lower for word in ['feature', 'request', 'enhancement']):
            return {"category": "feature_request", "confidence": 0.7}
        else:
            return {"category": "general", "confidence": 0.5}

@app.post("/predict-risk")
async def predict_risk(request: RiskRequest):
    """Predict user risk level based on behavior"""
    try:
        # Calculate risk score
        risk_score = 0.0
        
        # High number of overdue tasks
        if request.task_count > 0:
            overdue_ratio = request.overdue_tasks / request.task_count
            risk_score += overdue_ratio * 40
        
        # Slow completion time
        if request.avg_completion_time > 5:  # days
            risk_score += min((request.avg_completion_time - 5) * 5, 30)
        
        # Low login frequency
        if request.login_frequency < 3:  # per week
            risk_score += (3 - request.login_frequency) * 10
        
        risk_score = min(risk_score, 100)
        
        if risk_score >= 70:
            level = "high"
        elif risk_score >= 40:
            level = "medium"
        else:
            level = "low"
        
        return {
            "user_id": request.user_id,
            "risk_score": round(risk_score, 2),
            "risk_level": level,
            "factors": {
                "overdue_ratio": round(request.overdue_tasks / max(request.task_count, 1), 2),
                "avg_completion_days": round(request.avg_completion_time, 1),
                "login_frequency": round(request.login_frequency, 1)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutRequest):
    """Detect employee burnout risk"""
    try:
        burnout_score = 0.0
        
        # Excessive working hours
        if request.weekly_hours > 45:
            burnout_score += min((request.weekly_hours - 45) * 2, 40)
        
        # Low completion rate
        if request.task_completion_rate < 0.7:
            burnout_score += (0.7 - request.task_completion_rate) * 50
        
        # Missed deadlines
        burnout_score += min(request.missed_deadlines * 5, 30)
        
        # Overtime hours
        if request.overtime_hours > 10:
            burnout_score += min((request.overtime_hours - 10) * 2, 20)
        
        burnout_score = min(burnout_score, 100)
        
        if burnout_score >= 70:
            level = "critical"
        elif burnout_score >= 50:
            level = "high"
        elif burnout_score >= 30:
            level = "moderate"
        else:
            level = "low"
        
        return {
            "user_id": request.user_id,
            "burnout_score": round(burnout_score, 2),
            "burnout_level": level,
            "recommendations": get_burnout_recommendations(level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_burnout_recommendations(level: str):
    """Get recommendations based on burnout level"""
    recommendations = {
        "critical": [
            "Immediate intervention required",
            "Reduce workload by 30-50%",
            "Schedule mandatory time off",
            "Assign tasks to other team members"
        ],
        "high": [
            "Review and redistribute tasks",
            "Encourage regular breaks",
            "Monitor overtime hours closely",
            "Consider flexible work arrangements"
        ],
        "moderate": [
            "Monitor workload trends",
            "Ensure adequate resources",
            "Promote work-life balance"
        ],
        "low": [
            "Maintain current practices",
            "Continue regular check-ins"
        ]
    }
    return recommendations.get(level, [])

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {"status": "healthy", "models_loaded": len(models)}
```

## Frontend Integration

### React API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';
const ML_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_URL,
});

// Add token to requests
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
  getProfile: () => api.get('/auth/profile'),
};

export const taskAPI = {
  getMyTasks: () => api.get('/tasks/my-tasks'),
  getAllTasks: () => api.get('/tasks'),
  createTask: (taskData) => api.post('/tasks', taskData),
  updateTaskStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  deleteTask: (id) => api.delete(`/tasks/${id}`),
};

export const ticketAPI = {
  getTickets: () => api.get('/tickets'),
  createTicket: (ticketData) => api.post('/tickets', ticketData),
  updateTicket: (id, updates) => api.patch(`/tickets/${id}`, updates),
};

export const analyticsAPI = {
  getRiskPrediction: (userId) => api.get(`/analytics/risk/${userId}`),
  getBurnoutAnalysis: (userId) => api.get(`/analytics/burnout/${userId}`),
  getProjectInsights: () => api.get('/analytics/projects'),
};

export const mlAPI = {
  classifyTicket: (text) => axios.post(`${ML_URL}/classify-ticket`, { text }),
  predictRisk: (data) => axios.post(`${ML_URL}/predict-risk`, data),
  detectBurnout: (data) => axios.post(`${ML_URL}/detect-burnout`, data),
};

export default api;
```

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.js
import React, { useState, useEffect } from 'react';
import { taskAPI, mlAPI } from '../services/api';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const tasksResponse = await taskAPI.getMyTasks();
      setTasks(tasksResponse.data);
      
      // Calculate burnout metrics
      const completedTasks = tasksResponse.data.filter(t => t.status === 'done').length;
      const totalTasks = tasksResponse.data.length;
      const overdueTasks = tasksResponse.data.filter(t => 
        new Date(t.dueDate) < new Date() && t.status !== 'done'
      ).length;
      
      const burnoutResponse = await mlAPI.detectBurnout({
        user_id: localStorage.getItem('userId'),
        weekly_hours: 45,
        task_completion_rate: completedTasks / totalTasks,
        missed_deadlines: overdueTasks,
        overtime_hours: 5
      });
      
      setBurnoutData(burnoutResponse.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskAPI.updateTaskStatus(taskId, newStatus);
      fetchDashboardData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const getTasksByStatus = (status) => {
    return tasks.filter(task => task.status === status);
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {burnoutData && burnoutData.burnout_level !== 'low' && (
        <div className={`alert alert-${burnoutData.burnout_level}`}>
          <h3>Burnout Alert: {burnoutData.burnout_level.toUpperCase()}</h3>
          <p>Score: {burnoutData.burnout_score}/100</p>
          <ul>
            {burnoutData.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}
      
      <div className="kanban-board">
        <div className="kanban-column">
          <h2>To Do ({getTasksByStatus('todo').length})</h2>
          {getTasksByStatus('todo').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={handleStatusChange}
            />
          ))}
        </div>
        
        <div className="kanban-column">
          <h2>In Progress ({getTasksByStatus('in_progress').length})</h2>
          {getTasksByStatus('in_progress').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={handleStatusChange}
            />
          ))}
        </div>
        
        <div className="kanban-column">
          <h2>Done ({getTasksByStatus('done').length})</h2>
          {getTasksByStatus('done').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={handleStatusChange}
            />
          ))}
        </div>
      </div>
    </div>
  );
};

const TaskCard = ({ task, onStatusChange }) => {
  const isOverdue = new Date(task.dueDate) < new Date() && task.status !== 'done';
  
  return (
    <div className={`task-card ${isOverdue ? 'overdue' : ''}`}>
      <h3>{task.title}</h3>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <span>Due: {new Date(task.dueDate).toLocaleDateString()}</span>
      </div>
      <select 
        value={task.status} 
        onChange={(e) => onStatusChange(task._id, e.target.value)}
      >
        <option value="todo">To Do</option>
        <option value="in_progress">In Progress</option>
        <option value="done">Done</option>
      </select>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import { analyticsAPI, mlAPI } from '../services/api';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskAnalysis, setRiskAnalysis] = useState([]);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const usersResponse = await api.get('/users');
      const usersData = usersResponse.data;
      
      // Get risk predictions for all users
      const riskPromises = usersData.map(async (user) => {
        const tasks = await taskAPI.getAllTasks();
        const userTasks = tasks.data.filter(t => t.assignedTo === user._id);
        const overdueTasks = userTasks.filter(t => 
          new Date(t.dueDate) < new Date() && t.status !== 'done'
        ).length;
        
        const riskData = await mlAPI.predictRisk({
          user_id: user._id,
          task_count: userTasks.length,
          overdue_tasks: overdueTasks,
          avg_completion_time: 4.5,
          login_frequency: 5
        });
        
        return { ...user, risk: riskData.data };
      });
      
      const risks = await Promise.all(riskPromises);
      setUsers(usersData);
      setRiskAnalysis(risks);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const highRiskUsers = riskAnalysis.filter(u => u.risk.risk_level === 'high');

  return (
    <div className="admin-dashboard">
      <h1>Admin Analytics</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-value">{users.length}</p>
        </div>
        <div className="stat-card alert">
          <h3>High Risk Users</h3>
          <p className="stat-value">{highRiskUsers.length}</p>
        </div>
      </div>
      
      <h2>Risk Analysis</h2>
      <table className="analytics-table">
        <thead>
          <tr>
            <th>User</th>
            <th>Risk Level</th>
            <th>Risk Score</th>
            <th>Overdue Tasks</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {riskAnalysis.map(user => (
            <tr key={user._id} className={`risk-${user.risk.risk_level}`}>
              <td>{user.name}</td>
              <td>{user.risk.risk_level}</td>
              <td>{user.risk.risk_score}</td>
              <td>{user.risk.factors.overdue_ratio * 100}%</td>
              <td>
                <button onClick={() => handleUserIntervention(user._id)}>
                  Intervene
                </button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
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
    enum: ['user', 'admin'],
    default: 'user'
  },
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: {
    type: Date
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

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
  description: {
    type: String,
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
    type: Date,
    required: true
  },
  completedAt: {
    type: Date
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'access', 'feature_request', 'general'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in_progress', 'resolved', 'closed'],
    default: 'open'
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
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: {
    type: Date
  }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Workflows

### Complete Authentication Flow

```javascript
// frontend/src/contexts/AuthContext.js
import React, { createContext, useState, useEffect } from
