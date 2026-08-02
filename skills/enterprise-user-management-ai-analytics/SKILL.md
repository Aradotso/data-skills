---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket routing, and risk detection
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "show me how to implement task tracking with AI insights"
  - "create a user management dashboard with risk detection"
  - "how to build ticket classification with machine learning"
  - "implement role-based access control with AI features"
  - "set up predictive analytics for project management"
  - "configure JWT authentication for user management system"
---

# Enterprise User Management AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript/Python application that combines traditional user management with machine learning capabilities. It provides admin dashboards, user task tracking with Kanban boards, support ticket management, and AI-powered features including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

The system uses React for frontend, Node.js for backend APIs, MongoDB for data storage, FastAPI for ML services, and scikit-learn/River for machine learning models.

## Installation

### Prerequisites

```bash
# Required tools
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.4
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt

# Install frontend dependencies
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env)**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_users
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**
```env
MODEL_PATH=./models
DB_CONNECTION=mongodb://localhost:27017/enterprise_users
LOG_LEVEL=info
```

**Frontend (.env)**
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the System

```bash
# Terminal 1: Start MongoDB
mongod --dbpath /path/to/data

# Terminal 2: Start backend
cd backend
npm start

# Terminal 3: Start ML service
cd ml-service
uvicorn main:app --reload --port 8000

# Terminal 4: Start frontend
cd frontend
npm start
```

## Backend API Structure

### User Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// User registration
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);
    
    // Create user
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate token
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({
      message: 'User created successfully',
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// User login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    // Find user
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    // Verify password
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    // Generate token
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
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
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Get all tasks for user
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tasks', error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.user.userId,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    await task.save();
    await task.populate('assignedTo createdBy', 'username email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error creating task', error: error.message });
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
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    ).populate('assignedTo createdBy', 'username email');
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error updating task', error: error.message });
  }
});

// Track time on task
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body; // duration in seconds
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeTracked = (task.timeTracked || 0) + duration;
    await task.save();
    
    res.json({ timeTracked: task.timeTracked });
  } catch (error) {
    res.status(500).json({ message: 'Error tracking time', error: error.message });
  }
});

module.exports = router;
```

### Support Ticket Management

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authMiddleware, adminOnly } = require('../middleware/auth');

// Create support ticket
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    const ticket = new Ticket({
      title,
      description,
      priority: priority || 'medium',
      status: 'open',
      createdBy: req.user.userId
    });
    
    // Get AI classification
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        {
          title,
          description
        }
      );
      
      ticket.category = mlResponse.data.category;
      ticket.suggestedAssignee = mlResponse.data.suggestedAssignee;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    await ticket.save();
    await ticket.populate('createdBy', 'username email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Error creating ticket', error: error.message });
  }
});

// Get all tickets (admin)
router.get('/all', authMiddleware, adminOnly, async (req, res) => {
  try {
    const tickets = await Ticket.find()
      .populate('createdBy assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tickets', error: error.message });
  }
});

// Get user's tickets
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.userId })
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tickets', error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI ML Service Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
import joblib
import os

app = FastAPI(title="Enterprise ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Request/Response models
class TicketRequest(BaseModel):
    title: str
    description: str

class TicketResponse(BaseModel):
    category: str
    suggestedAssignee: Optional[str]
    confidence: float

class RiskRequest(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    accessPatterns: List[str]
    lastActivityHours: int

class RiskResponse(BaseModel):
    riskLevel: str
    riskScore: float
    anomalyDetected: bool
    recommendations: List[str]

class BurnoutRequest(BaseModel):
    userId: str
    hoursWorked: float
    tasksCompleted: int
    tasksOverdue: int
    avgTaskDuration: float

class BurnoutResponse(BaseModel):
    burnoutRisk: str
    score: float
    recommendations: List[str]

@app.get("/")
async def root():
    return {"message": "Enterprise ML Service Active"}

@app.post("/classify-ticket", response_model=TicketResponse)
async def classify_ticket(request: TicketRequest):
    """Classify support ticket and suggest assignee"""
    try:
        # Simple keyword-based classification
        text = (request.title + " " + request.description).lower()
        
        categories = {
            'technical': ['bug', 'error', 'crash', 'issue', 'not working'],
            'account': ['password', 'login', 'access', 'permission'],
            'feature': ['request', 'need', 'add', 'new feature'],
            'general': []
        }
        
        scores = {}
        for category, keywords in categories.items():
            score = sum(1 for keyword in keywords if keyword in text)
            scores[category] = score
        
        category = max(scores, key=scores.get) if max(scores.values()) > 0 else 'general'
        confidence = max(scores.values()) / (len(text.split()) + 1)
        
        # Suggest assignee based on category
        assignees = {
            'technical': 'tech-support',
            'account': 'account-manager',
            'feature': 'product-team',
            'general': 'general-support'
        }
        
        return TicketResponse(
            category=category,
            suggestedAssignee=assignees.get(category),
            confidence=min(confidence, 1.0)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk", response_model=RiskResponse)
async def predict_risk(request: RiskRequest):
    """Predict user risk based on behavior"""
    try:
        # Calculate risk score
        risk_score = 0.0
        recommendations = []
        
        # Failed login attempts
        if request.failedLogins > 3:
            risk_score += 0.3
            recommendations.append("Multiple failed login attempts detected")
        
        # Unusual access patterns
        if len(set(request.accessPatterns)) > 5:
            risk_score += 0.2
            recommendations.append("Unusual access pattern detected")
        
        # Inactive account
        if request.lastActivityHours > 720:  # 30 days
            risk_score += 0.1
            recommendations.append("Account inactive for extended period")
        
        # Excessive login attempts
        if request.loginAttempts > 20:
            risk_score += 0.2
            recommendations.append("Excessive login activity")
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = "high"
        elif risk_score >= 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        # Anomaly detection
        anomaly_detected = risk_score >= 0.5
        
        if not recommendations:
            recommendations.append("No significant risks detected")
        
        return RiskResponse(
            riskLevel=risk_level,
            riskScore=min(risk_score, 1.0),
            anomalyDetected=anomaly_detected,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout", response_model=BurnoutResponse)
async def detect_burnout(request: BurnoutRequest):
    """Detect employee burnout risk"""
    try:
        score = 0.0
        recommendations = []
        
        # Hours worked analysis
        if request.hoursWorked > 50:
            score += 0.3
            recommendations.append("Excessive working hours detected")
        elif request.hoursWorked > 45:
            score += 0.15
            recommendations.append("High working hours")
        
        # Overdue tasks
        overdue_ratio = request.tasksOverdue / max(request.tasksCompleted, 1)
        if overdue_ratio > 0.3:
            score += 0.3
            recommendations.append("High ratio of overdue tasks")
        elif overdue_ratio > 0.15:
            score += 0.15
        
        # Task completion rate
        if request.avgTaskDuration > 8:  # hours
            score += 0.2
            recommendations.append("Tasks taking longer than average")
        
        # Workload
        total_tasks = request.tasksCompleted + request.tasksOverdue
        if total_tasks > 30:
            score += 0.2
            recommendations.append("Heavy workload detected")
        
        # Determine burnout risk
        if score >= 0.7:
            burnout_risk = "high"
            recommendations.append("Immediate intervention recommended")
        elif score >= 0.4:
            burnout_risk = "medium"
            recommendations.append("Monitor workload closely")
        else:
            burnout_risk = "low"
        
        if not recommendations or burnout_risk == "low":
            recommendations.append("Workload appears manageable")
        
        return BurnoutResponse(
            burnoutRisk=burnout_risk,
            score=min(score, 1.0),
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Frontend React Implementation

### API Service Layer

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Create axios instance
const api = axios.create({
  baseURL: API_URL,
});

// Add auth token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auth API
export const authAPI = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (userData) => api.post('/auth/register', userData),
  logout: () => {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  }
};

// User API
export const userAPI = {
  getAll: () => api.get('/users'),
  getById: (id) => api.get(`/users/${id}`),
  create: (userData) => api.post('/users', userData),
  update: (id, userData) => api.patch(`/users/${id}`, userData),
  delete: (id) => api.delete(`/users/${id}`)
};

// Task API
export const taskAPI = {
  getAll: () => api.get('/tasks'),
  create: (taskData) => api.post('/tasks', taskData),
  updateStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  trackTime: (id, duration) => api.post(`/tasks/${id}/time`, { duration }),
  delete: (id) => api.delete(`/tasks/${id}`)
};

// Ticket API
export const ticketAPI = {
  getAll: () => api.get('/tickets/all'),
  getUserTickets: () => api.get('/tickets'),
  create: (ticketData) => api.post('/tickets', ticketData),
  update: (id, updates) => api.patch(`/tickets/${id}`, updates)
};

// ML API
const mlApi = axios.create({
  baseURL: ML_API_URL,
});

export const mlAPI = {
  classifyTicket: (title, description) => 
    mlApi.post('/classify-ticket', { title, description }),
  
  predictRisk: (userData) => 
    mlApi.post('/predict-risk', userData),
  
  detectBurnout: (workloadData) => 
    mlApi.post('/detect-burnout', workloadData)
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
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getAll();
      const grouped = groupTasksByStatus(response.data);
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const groupTasksByStatus = (taskList) => {
    return {
      todo: taskList.filter(t => t.status === 'todo'),
      inprogress: taskList.filter(t => t.status === 'inprogress'),
      done: taskList.filter(t => t.status === 'done')
    };
  };

  const handleDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');

    if (currentStatus === newStatus) return;

    try {
      await taskAPI.updateStatus(taskId, newStatus);
      fetchTasks();
    } catch (error) {
      console.error('Error updating task status:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const TaskCard = ({ task }) => (
    <div 
      className="task-card"
      draggable
      onDragStart={(e) => handleDragStart(e, task._id, task.status)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        {task.timeTracked && (
          <span className="time">{Math.floor(task.timeTracked / 60)}m</span>
        )}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      <div 
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'todo')}
        onDragOver={handleDragOver}
      >
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>

      <div 
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'inprogress')}
        onDragOver={handleDragOver}
      >
        <h3>In Progress ({tasks.inprogress.length})</h3>
        {tasks.inprogress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>

      <div 
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'done')}
        onDragOver={handleDragOver}
      >
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { mlAPI, userAPI } from '../services/api';
import './AIAnalytics.css';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskAnalysis: null,
    burnoutAnalysis: null,
    loading: true
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch user data for analysis
      const userResponse = await userAPI.getById(userId);
      const userData = userResponse.data;

      // Get risk prediction
      const riskResponse = await mlAPI.predictRisk({
        userId: userData._id,
        loginAttempts: userData.loginAttempts || 0,
        failedLogins: userData.failedLogins || 0,
        accessPatterns: userData.accessPatterns || [],
        lastActivityHours: calculateHoursSinceActivity(userData.lastActivity)
      });

      // Get burnout detection
      const burnoutResponse = await mlAPI.detectBurnout({
        userId: userData._id,
        hoursWorked: userData.hoursWorked || 0,
        tasksCompleted: userData.tasksCompleted || 0,
        tasksOverdue: userData.tasksOverdue || 0,
        avgTaskDuration: userData.avgTaskDuration || 0
      });

      setAnalytics({
        riskAnalysis: riskResponse.data,
        burnoutAnalysis: burnoutResponse.data,
        loading: false
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
      setAnalytics(prev => ({ ...prev, loading: false }));
    }
  };

  const calculateHoursSinceActivity = (lastActivity) => {
    if (!lastActivity) return 0;
    const diff = Date.now() - new Date(lastActivity).getTime();
    return Math.floor(diff / (1000 * 60 * 60));
  };

  const getRiskColor = (level) => {
    const colors = { low: 'green', medium: 'orange', high: 'red' };
    return colors[level] || 'gray';
  };

  if (analytics.loading) {
    return <div className="analytics-loading">Loading AI analytics...</div>;
  }

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Analytics</h2>

      {/* Risk Analysis */}
      {analytics.riskAnalysis && (
        <div className="analytics-card">
          <h3>Security Risk Analysis</h3>
          <div className="risk-meter">
            <div 
              className="risk-indicator"
              style={{ 
                backgroundColor: getRiskColor(analytics.riskAnalysis.riskLevel),
                width: `${analytics.riskAnalysis.riskScore * 100}%`
              }}
            />
          </div>
          <div className="risk-details">
            <p><strong>Risk Level:</strong> {analytics.riskAnalysis.riskLevel.toUpperCase()}</p>
            <p><strong>Risk Score:</strong> {(analytics.riskAnalysis.riskScore * 100).toFixed(1)}%</p>
            <p><strong>Anomaly Detected:</strong> {analytics.riskAnalysis.anomalyDetected ? 'Yes' : 'No'}</p>
          </div>
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {analytics.riskAnalysis.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        </div>
      )}

      {/* Burnout Analysis */}
      {analytics.burnoutAnalysis && (
        <div className="analytics-card">
          <h3>Burnout Risk Detection</h3>
          <div className="burnout-meter">
            <div 
              className="burnout-indicator"
              style={{ 
                backgroundColor: getRiskColor(analytics.burnoutAnalysis.burnoutRisk),
                width: `${analytics.burnoutAnalysis.score * 100}%`
              }}
            />
          </div>
          <div className="burnout-details">
            <p><strong>Burnout Risk:</strong> {analytics.burnoutAnalysis.burnoutRisk.toUpperCase()}</p>
            <p><strong>Score:</strong> {(analytics.burnoutAnalysis.score * 100).toFixed(1)}%</p>
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

### MongoDB Schema Definitions

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

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
    type:
