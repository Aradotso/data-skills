---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management capabilities
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics for user management"
  - "show me how to implement task tracking with AI insights"
  - "configure the user management system with ML service"
  - "set up JWT authentication for enterprise users"
  - "implement AI-based ticket classification and routing"
  - "create a user dashboard with AI analytics"
  - "add risk prediction to the user management app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack JavaScript/Node.js application that provides centralized user, task, and ticket management with integrated machine learning capabilities. It combines React.js frontend, Node.js/Express backend, MongoDB database, and FastAPI ML service to deliver intelligent insights including risk detection, anomaly detection, burnout analysis, and predictive analytics.

**Key capabilities:**
- Role-based user management (Admin/User)
- Task tracking with Kanban board
- Support ticket system with AI classification
- Real-time AI analytics and predictions
- JWT-based authentication
- Performance monitoring and audit logs

## Installation

### Prerequisites
```bash
# Required software
node -v  # v14+ required
npm -v   # v6+ required
python --version  # Python 3.8+ for ML service
mongod --version  # MongoDB 4.4+
```

### Complete Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
echo "MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your_jwt_secret_key_here
PORT=5000
ML_SERVICE_URL=http://localhost:8000" > .env
npm start

# ML Service setup (new terminal)
cd ml-service
pip install -r requirements.txt
echo "MODEL_PATH=./models
LOG_LEVEL=info" > .env
uvicorn main:app --reload --port 8000

# Frontend setup (new terminal)
cd frontend
npm install
echo "REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000" > .env
npm start
```

## Architecture

The system consists of three main services:

1. **Frontend** (React.js, port 3000) - User interface
2. **Backend** (Node.js/Express, port 5000) - API and business logic
3. **ML Service** (FastAPI, port 8000) - AI/ML predictions

## Backend API Development

### User Authentication

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
    const { email, password, name, role } = req.body;
    
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
      email,
      password: hashedPassword,
      name,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate JWT token
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.status(201).json({
      token,
      user: {
        id: user._id,
        email: user.email,
        name: user.name,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({
      token,
      user: {
        id: user._id,
        email: user.email,
        name: user.name,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'No token, authorization denied' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token is not valid' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

// Get all tasks for user
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ 
      assignedTo: req.user.userId 
    }).populate('assignedBy', 'name email');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Create task (admin only)
router.post('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.userId,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    await task.save();
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const validStatuses = ['todo', 'inprogress', 'done'];
    
    if (!validStatuses.includes(status)) {
      return res.status(400).json({ message: 'Invalid status' });
    }
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.userId },
      { status, updatedAt: Date.now() },
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

### Support Ticket System

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authMiddleware } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for automatic classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      { title, description }
    );
    
    const ticket = new Ticket({
      title,
      description,
      priority: priority || 'medium',
      category: mlResponse.data.category,
      assignedDepartment: mlResponse.data.department,
      createdBy: req.user.userId,
      status: 'open'
    });
    
    await ticket.save();
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ 
      createdBy: req.user.userId 
    }).sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

## ML Service Development

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Dict
import os

app = FastAPI(title="Enterprise UMS ML Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Request/Response models
class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    department: str
    confidence: float

class RiskPredictionRequest(BaseModel):
    user_id: str
    recent_activities: List[Dict]
    task_completion_rate: float
    avg_response_time: float

class RiskPredictionResponse(BaseModel):
    risk_level: str
    risk_score: float
    factors: List[str]

# Load models (implement lazy loading in production)
MODEL_PATH = os.getenv('MODEL_PATH', './models')

# Ticket classification endpoint
@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    try:
        # Simple rule-based classification (replace with ML model)
        text = f"{request.title} {request.description}".lower()
        
        # Category classification
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            department = 'IT'
        elif any(word in text for word in ['access', 'permission', 'login', 'password']):
            category = 'access'
            department = 'Security'
        elif any(word in text for word in ['payment', 'invoice', 'billing']):
            category = 'billing'
            department = 'Finance'
        else:
            category = 'general'
            department = 'Support'
        
        return TicketClassificationResponse(
            category=category,
            department=department,
            confidence=0.85
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Risk prediction endpoint
@app.post("/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    try:
        risk_factors = []
        risk_score = 0.0
        
        # Analyze task completion rate
        if request.task_completion_rate < 0.5:
            risk_score += 0.3
            risk_factors.append("Low task completion rate")
        
        # Analyze response time
        if request.avg_response_time > 48:  # hours
            risk_score += 0.2
            risk_factors.append("Slow response time")
        
        # Analyze activity patterns
        if len(request.recent_activities) < 5:
            risk_score += 0.2
            risk_factors.append("Low activity")
        
        # Determine risk level
        if risk_score < 0.3:
            risk_level = "low"
        elif risk_score < 0.6:
            risk_level = "medium"
        else:
            risk_level = "high"
        
        return RiskPredictionResponse(
            risk_level=risk_level,
            risk_score=min(risk_score, 1.0),
            factors=risk_factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout detection
class BurnoutRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_work_hours: float
    overtime_hours: float

class BurnoutResponse(BaseModel):
    burnout_risk: str
    score: float
    recommendations: List[str]

@app.post("/detect-burnout", response_model=BurnoutResponse)
async def detect_burnout(request: BurnoutRequest):
    try:
        score = 0.0
        recommendations = []
        
        # Workload analysis
        if request.tasks_assigned > 20:
            score += 0.25
            recommendations.append("Reduce task assignment")
        
        # Working hours analysis
        if request.avg_work_hours > 9:
            score += 0.3
            recommendations.append("Monitor working hours")
        
        if request.overtime_hours > 10:
            score += 0.25
            recommendations.append("Limit overtime")
        
        # Completion rate
        completion_rate = request.tasks_completed / max(request.tasks_assigned, 1)
        if completion_rate < 0.5:
            score += 0.2
            recommendations.append("Review task difficulty")
        
        # Determine burnout risk
        if score < 0.3:
            burnout_risk = "low"
        elif score < 0.6:
            burnout_risk = "moderate"
        else:
            burnout_risk = "high"
        
        return BurnoutResponse(
            burnout_risk=burnout_risk,
            score=min(score, 1.0),
            recommendations=recommendations or ["Continue current pace"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Frontend Development

### API Service Layer

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

// Create axios instance
const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor to add auth token
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
  getCurrentUser: () => api.get('/auth/me'),
};

// User API
export const userAPI = {
  getAllUsers: () => api.get('/users'),
  getUser: (id) => api.get(`/users/${id}`),
  createUser: (userData) => api.post('/users', userData),
  updateUser: (id, userData) => api.put(`/users/${id}`, userData),
  deleteUser: (id) => api.delete(`/users/${id}`),
};

// Task API
export const taskAPI = {
  getTasks: () => api.get('/tasks'),
  createTask: (taskData) => api.post('/tasks', taskData),
  updateTaskStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  deleteTask: (id) => api.delete(`/tasks/${id}`),
};

// Ticket API
export const ticketAPI = {
  createTicket: (ticketData) => api.post('/tickets', ticketData),
  getMyTickets: () => api.get('/tickets/my-tickets'),
  getAllTickets: () => api.get('/tickets'),
  updateTicket: (id, data) => api.put(`/tickets/${id}`, data),
};

// ML API
export const mlAPI = {
  classifyTicket: (data) => axios.post(`${ML_URL}/classify-ticket`, data),
  predictRisk: (data) => axios.post(`${ML_URL}/predict-risk`, data),
  detectBurnout: (data) => axios.post(`${ML_URL}/detect-burnout`, data),
};

export default api;
```

### React Context for Authentication

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect, useContext } from 'react';
import { authAPI } from '../services/api';

const AuthContext = createContext();

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    checkAuth();
  }, []);

  const checkAuth = async () => {
    const token = localStorage.getItem('token');
    if (token) {
      try {
        const response = await authAPI.getCurrentUser();
        setUser(response.data.user);
      } catch (error) {
        localStorage.removeItem('token');
      }
    }
    setLoading(false);
  };

  const login = async (credentials) => {
    const response = await authAPI.login(credentials);
    localStorage.setItem('token', response.data.token);
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  const register = async (userData) => {
    const response = await authAPI.register(userData);
    localStorage.setItem('token', response.data.token);
    setUser(response.data.user);
    return response.data;
  };

  const value = {
    user,
    login,
    logout,
    register,
    isAuthenticated: !!user,
    isAdmin: user?.role === 'admin',
  };

  return (
    <AuthContext.Provider value={value}>
      {!loading && children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getTasks();
      const tasksByStatus = {
        todo: [],
        inprogress: [],
        done: []
      };
      
      response.data.forEach(task => {
        tasksByStatus[task.status].push(task);
      });
      
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskAPI.updateTaskStatus(taskId, newStatus);
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <span className="due-date">
          {new Date(task.dueDate).toLocaleDateString()}
        </span>
      </div>
      <select
        value={task.status}
        onChange={(e) => handleStatusChange(task._id, e.target.value)}
        className="status-select"
      >
        <option value="todo">To Do</option>
        <option value="inprogress">In Progress</option>
        <option value="done">Done</option>
      </select>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do ({tasks.todo.length})</h3>
        <div className="task-list">
          {tasks.todo.map(task => (
            <TaskCard key={task._id} task={task} />
          ))}
        </div>
      </div>
      
      <div className="kanban-column">
        <h3>In Progress ({tasks.inprogress.length})</h3>
        <div className="task-list">
          {tasks.inprogress.map(task => (
            <TaskCard key={task._id} task={task} />
          ))}
        </div>
      </div>
      
      <div className="kanban-column">
        <h3>Done ({tasks.done.length})</h3>
        <div className="task-list">
          {tasks.done.map(task => (
            <TaskCard key={task._id} task={task} />
          ))}
        </div>
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import { mlAPI, userAPI, taskAPI } from '../services/api';

const AIAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    riskPrediction: null,
    burnoutAnalysis: null,
    loading: true
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const tasksResponse = await taskAPI.getTasks();
      const tasks = tasksResponse.data;
      
      const completedTasks = tasks.filter(t => t.status === 'done').length;
      const totalTasks = tasks.length;
      
      // Get risk prediction
      const riskData = {
        user_id: 'current_user',
        recent_activities: tasks.slice(-10),
        task_completion_rate: totalTasks > 0 ? completedTasks / totalTasks : 0,
        avg_response_time: 24
      };
      
      const riskResponse = await mlAPI.predictRisk(riskData);
      
      // Get burnout analysis
      const burnoutData = {
        user_id: 'current_user',
        tasks_assigned: totalTasks,
        tasks_completed: completedTasks,
        avg_work_hours: 8.5,
        overtime_hours: 5
      };
      
      const burnoutResponse = await mlAPI.detectBurnout(burnoutData);
      
      setAnalytics({
        riskPrediction: riskResponse.data,
        burnoutAnalysis: burnoutResponse.data,
        loading: false
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
      setAnalytics(prev => ({ ...prev, loading: false }));
    }
  };

  if (analytics.loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Analytics</h2>
      
      {analytics.riskPrediction && (
        <div className="analytics-card">
          <h3>Risk Assessment</h3>
          <div className={`risk-level ${analytics.riskPrediction.risk_level}`}>
            Risk Level: {analytics.riskPrediction.risk_level.toUpperCase()}
          </div>
          <div className="risk-score">
            Score: {(analytics.riskPrediction.risk_score * 100).toFixed(1)}%
          </div>
          <div className="risk-factors">
            <h4>Risk Factors:</h4>
            <ul>
              {analytics.riskPrediction.factors.map((factor, idx) => (
                <li key={idx}>{factor}</li>
              ))}
            </ul>
          </div>
        </div>
      )}
      
      {analytics.burnoutAnalysis && (
        <div className="analytics-card">
          <h3>Burnout Detection</h3>
          <div className={`burnout-risk ${analytics.burnoutAnalysis.burnout_risk}`}>
            Burnout Risk: {analytics.burnoutAnalysis.burnout_risk.toUpperCase()}
          </div>
          <div className="burnout-score">
            Score: {(analytics.burnoutAnalysis.score * 100).toFixed(1)}%
          </div>
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {analytics.burnoutAnalysis.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        </div>
      )}
      
      <button onClick={fetchAnalytics} className="refresh-btn">
        Refresh Analytics
      </button>
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

const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true
  },
  password: {
    type: String,
    required: true
  },
  name: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date
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
  description: String,
  status: {
    type: String,
    enum: ['todo', 'inprogress', 'done'],
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
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: Date,
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
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
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  category: String,
  assignedDepartment: String,
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
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_ums

# JWT
JWT_SECRET=your_secure_jwt_secret_minimum_32_characters

# Server
PORT=5000
NODE_ENV=development

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP
