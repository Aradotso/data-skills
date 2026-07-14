---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, risk detection, and productivity insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement task tracking with burnout detection"
  - "build user management dashboard with ML insights"
  - "add AI ticket classification system"
  - "create role-based user management with analytics"
  - "deploy user management system with FastAPI ML service"
  - "configure JWT authentication for enterprise users"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that provides centralized user, task, and ticket management with integrated machine learning capabilities. The system includes:

- **Backend**: Node.js REST API with JWT authentication and MongoDB
- **Frontend**: React.js dashboard with Kanban boards and time tracking
- **ML Service**: FastAPI-based AI engine for risk detection, anomaly detection, burnout analysis, and predictive insights
- **Role-based Access**: Admin and user roles with different permissions

## Installation

### Prerequisites

```bash
# Node.js 14+
node --version

# Python 3.8+ for ML service
python --version

# MongoDB running locally or cloud instance
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
NODE_ENV=development
EOF

# Start backend server
npm start
# Runs on http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file for ML service
cat > .env << EOF
API_KEY=${ML_API_KEY}
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
# Runs on http://localhost:8000
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
# Runs on http://localhost:3000
```

## Architecture

### Backend API Structure

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Connect to MongoDB
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log('MongoDB Connected'))
  .catch(err => console.error('MongoDB connection error:', err));

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/tasks', require('./routes/tasks'));
app.use('/api/tickets', require('./routes/tickets'));
app.use('/api/analytics', require('./routes/analytics'));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### User Authentication Middleware

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
    req.user = decoded.user;
    next();
  } catch (err) {
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

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  workloadScore: { type: Number, default: 0 },
  riskScore: { type: Number, default: 0 }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
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
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['todo', 'in-progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  dueDate: Date,
  estimatedHours: Number,
  actualHours: { type: Number, default: 0 },
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  ticketId: { type: String, unique: true },
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { 
    type: String, 
    enum: ['technical', 'access', 'bug', 'feature', 'other'] 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: {
    category: String,
    confidence: Number,
    suggestedPriority: String
  },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

// Auto-generate ticket ID
ticketSchema.pre('save', async function(next) {
  if (!this.ticketId) {
    const count = await this.constructor.countDocuments();
    this.ticketId = `TKT-${String(count + 1).padStart(5, '0')}`;
  }
  next();
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Authentication API

### Register User

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    // Check if user exists
    let user = await User.findOne({ email });
    if (user) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Create new user
    user = new User({ name, email, password, role });
    await user.save();
    
    // Generate JWT token
    const payload = {
      user: {
        id: user.id,
        role: user.role
      }
    };
    
    const token = jwt.sign(payload, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });
    
    res.json({ token, user: { id: user.id, name: user.name, role: user.role } });
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Server error' });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    // Check user exists
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    // Verify password
    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    // Update last login
    user.lastLogin = new Date();
    await user.save();
    
    // Generate token
    const payload = {
      user: {
        id: user.id,
        role: user.role
      }
    };
    
    const token = jwt.sign(payload, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });
    
    res.json({ token, user: { id: user.id, name: user.name, role: user.role } });
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Server error' });
  }
});

module.exports = router;
```

## Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

// Get all tasks (filtered by user role)
router.get('/', authMiddleware, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user.id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Server error' });
  }
});

// Create new task
router.post('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    
    await task.save();
    await task.populate('assignedTo', 'name email');
    
    res.status(201).json(task);
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Server error' });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.status = status;
    task.updatedAt = new Date();
    await task.save();
    
    res.json(task);
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Server error' });
  }
});

// Update time tracking
router.patch('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { actualHours } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { actualHours, updatedAt: new Date() },
      { new: true }
    );
    
    res.json(task);
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Server error' });
  }
});

module.exports = router;
```

## ML Service - AI Analytics

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from datetime import datetime
import joblib

app = FastAPI(title="Enterprise User Management ML Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models (load pre-trained or initialize)
try:
    risk_model = joblib.load('./models/risk_model.pkl')
    anomaly_model = joblib.load('./models/anomaly_model.pkl')
except:
    risk_model = None
    anomaly_model = None

# Request models
class TicketData(BaseModel):
    title: str
    description: str

class UserBehaviorData(BaseModel):
    userId: str
    loginCount: int
    taskCompletionRate: float
    avgResponseTime: float
    failedLogins: int
    lastLogin: str

class WorkloadData(BaseModel):
    userId: str
    tasksAssigned: int
    tasksCompleted: int
    totalHours: float
    overtimeHours: float
    avgTaskDuration: float

@app.get("/")
def root():
    return {"message": "Enterprise User Management ML Service", "status": "running"}

@app.get("/health")
def health_check():
    return {"status": "healthy", "timestamp": datetime.now().isoformat()}
```

### AI Ticket Classification

```python
# ml-service/main.py (continued)
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import re

# Initialize ticket classifier
ticket_vectorizer = TfidfVectorizer(max_features=100)
ticket_classifier = MultinomialNB()

# Categories mapping
TICKET_CATEGORIES = {
    0: 'technical',
    1: 'access',
    2: 'bug',
    3: 'feature',
    4: 'other'
}

PRIORITY_MAP = {
    'low': 1,
    'medium': 2,
    'high': 3,
    'critical': 4
}

def preprocess_text(text):
    """Clean and preprocess text"""
    text = text.lower()
    text = re.sub(r'[^a-zA-Z0-9\s]', '', text)
    return text

@app.post("/classify-ticket")
def classify_ticket(ticket: TicketData):
    """AI-based ticket classification and priority suggestion"""
    try:
        # Combine title and description
        full_text = f"{ticket.title} {ticket.description}"
        processed_text = preprocess_text(full_text)
        
        # Simple rule-based classification (can be replaced with trained model)
        category = 'other'
        confidence = 0.6
        priority = 'medium'
        
        # Keywords for classification
        if any(word in processed_text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'bug'
            priority = 'high'
            confidence = 0.85
        elif any(word in processed_text for word in ['access', 'login', 'permission', 'password']):
            category = 'access'
            priority = 'medium'
            confidence = 0.80
        elif any(word in processed_text for word in ['feature', 'enhancement', 'request', 'add']):
            category = 'feature'
            priority = 'low'
            confidence = 0.75
        elif any(word in processed_text for word in ['urgent', 'critical', 'production', 'down']):
            priority = 'critical'
            confidence = 0.90
        elif any(word in processed_text for word in ['server', 'network', 'database', 'api']):
            category = 'technical'
            priority = 'high'
            confidence = 0.80
        
        return {
            "category": category,
            "confidence": confidence,
            "suggestedPriority": priority,
            "keywords": processed_text.split()[:5]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Risk Detection

```python
# ml-service/main.py (continued)
@app.post("/detect-risk")
def detect_risk(data: UserBehaviorData):
    """Detect potential security risks based on user behavior"""
    try:
        risk_score = 0.0
        risk_factors = []
        
        # Failed login analysis
        if data.failedLogins > 5:
            risk_score += 0.3
            risk_factors.append("Multiple failed login attempts")
        
        # Low task completion rate
        if data.taskCompletionRate < 0.5:
            risk_score += 0.2
            risk_factors.append("Low task completion rate")
        
        # Unusual response time
        if data.avgResponseTime > 3600:  # More than 1 hour
            risk_score += 0.15
            risk_factors.append("Unusually high response time")
        
        # Irregular login patterns
        if data.loginCount > 50 or data.loginCount == 0:
            risk_score += 0.2
            risk_factors.append("Irregular login pattern")
        
        # Cap risk score at 1.0
        risk_score = min(risk_score, 1.0)
        
        risk_level = "low"
        if risk_score > 0.7:
            risk_level = "critical"
        elif risk_score > 0.5:
            risk_level = "high"
        elif risk_score > 0.3:
            risk_level = "medium"
        
        return {
            "userId": data.userId,
            "riskScore": round(risk_score, 2),
            "riskLevel": risk_level,
            "riskFactors": risk_factors,
            "recommendation": "Review user activity" if risk_score > 0.5 else "Normal"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Burnout Detection

```python
# ml-service/main.py (continued)
@app.post("/detect-burnout")
def detect_burnout(data: WorkloadData):
    """Analyze workload and detect potential employee burnout"""
    try:
        burnout_score = 0.0
        burnout_factors = []
        
        # Calculate workload ratio
        completion_rate = data.tasksCompleted / max(data.tasksAssigned, 1)
        
        # High number of assigned tasks
        if data.tasksAssigned > 20:
            burnout_score += 0.25
            burnout_factors.append("High task volume")
        
        # Low completion rate
        if completion_rate < 0.6:
            burnout_score += 0.2
            burnout_factors.append("Low task completion rate")
        
        # Excessive overtime
        if data.overtimeHours > 20:
            burnout_score += 0.3
            burnout_factors.append("Excessive overtime hours")
        
        # Long average task duration
        if data.avgTaskDuration > 8:
            burnout_score += 0.15
            burnout_factors.append("Extended task durations")
        
        # Total hours worked
        if data.totalHours > 60:
            burnout_score += 0.2
            burnout_factors.append("Excessive total work hours")
        
        burnout_score = min(burnout_score, 1.0)
        
        burnout_level = "low"
        if burnout_score > 0.7:
            burnout_level = "critical"
        elif burnout_score > 0.5:
            burnout_level = "high"
        elif burnout_score > 0.3:
            burnout_level = "medium"
        
        recommendations = []
        if burnout_score > 0.5:
            recommendations.append("Consider redistributing workload")
            recommendations.append("Schedule time off or breaks")
        if data.overtimeHours > 20:
            recommendations.append("Reduce overtime hours")
        
        return {
            "userId": data.userId,
            "burnoutScore": round(burnout_score, 2),
            "burnoutLevel": burnout_level,
            "burnoutFactors": burnout_factors,
            "recommendations": recommendations,
            "workloadMetrics": {
                "completionRate": round(completion_rate, 2),
                "averageHoursPerTask": round(data.totalHours / max(data.tasksCompleted, 1), 2)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Anomaly Detection

```python
# ml-service/main.py (continued)
from river import anomaly, preprocessing

# Initialize online learning anomaly detector
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)
scaler = preprocessing.StandardScaler()

@app.post("/detect-anomaly")
def detect_anomaly(data: UserBehaviorData):
    """Detect anomalies in user behavior using online learning"""
    try:
        # Prepare features
        features = {
            'loginCount': data.loginCount,
            'taskCompletionRate': data.taskCompletionRate,
            'avgResponseTime': data.avgResponseTime,
            'failedLogins': data.failedLogins
        }
        
        # Scale features
        scaled_features = scaler.learn_one(features).transform_one(features)
        
        # Get anomaly score
        anomaly_score = anomaly_detector.score_one(scaled_features)
        
        # Update model
        anomaly_detector.learn_one(scaled_features)
        
        is_anomaly = anomaly_score > 0.6
        
        return {
            "userId": data.userId,
            "anomalyScore": round(float(anomaly_score), 3),
            "isAnomaly": is_anomaly,
            "severity": "high" if anomaly_score > 0.8 else "medium" if anomaly_score > 0.6 else "low",
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Integration

### React API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Create axios instance
const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
api.interceptors.request.use(
  config => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => Promise.reject(error)
);

// Auth API
export const authAPI = {
  login: (email, password) => 
    api.post('/api/auth/login', { email, password }),
  
  register: (userData) => 
    api.post('/api/auth/register', userData),
  
  getCurrentUser: () => 
    api.get('/api/auth/me')
};

// User API
export const userAPI = {
  getAll: () => api.get('/api/users'),
  getById: (id) => api.get(`/api/users/${id}`),
  create: (userData) => api.post('/api/users', userData),
  update: (id, userData) => api.patch(`/api/users/${id}`, userData),
  delete: (id) => api.delete(`/api/users/${id}`)
};

// Task API
export const taskAPI = {
  getAll: () => api.get('/api/tasks'),
  getById: (id) => api.get(`/api/tasks/${id}`),
  create: (taskData) => api.post('/api/tasks', taskData),
  updateStatus: (id, status) => 
    api.patch(`/api/tasks/${id}/status`, { status }),
  updateTime: (id, actualHours) => 
    api.patch(`/api/tasks/${id}/time`, { actualHours }),
  delete: (id) => api.delete(`/api/tasks/${id}`)
};

// Ticket API
export const ticketAPI = {
  getAll: () => api.get('/api/tickets'),
  getById: (id) => api.get(`/api/tickets/${id}`),
  create: (ticketData) => api.post('/api/tickets', ticketData),
  update: (id, ticketData) => api.patch(`/api/tickets/${id}`, ticketData),
  delete: (id) => api.delete(`/api/tickets/${id}`)
};

// ML API
export const mlAPI = {
  classifyTicket: (title, description) =>
    axios.post(`${ML_API_URL}/classify-ticket`, { title, description }),
  
  detectRisk: (userData) =>
    axios.post(`${ML_API_URL}/detect-risk`, userData),
  
  detectBurnout: (workloadData) =>
    axios.post(`${ML_API_URL}/detect-burnout`, workloadData),
  
  detectAnomaly: (behaviorData) =>
    axios.post(`${ML_API_URL}/detect-anomaly`, behaviorData)
};

export default api;
```

### React Kanban Board Component

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
      const response = await taskAPI.getAll();
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
      await taskAPI.updateStatus(taskId, newStatus);
      fetchTasks(); // Refresh tasks
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
        <span className="assigned-to">{task.assignedTo?.name}</span>
      </div>
      <div className="task-actions">
        {task.status !== 'todo' && (
          <button onClick={() => handleStatusChange(task._id, 'todo')}>
            ← To Do
          </button>
        )}
        {task.status !== 'in-progress' && (
          <button onClick={() => handleStatusChange(task._id, 'in-progress')}>
            → In Progress
          </button>
        )}
        {task.status !== 'done' && (
          <button onClick={() => handleStatusChange(task._id, 'done')}>
            ✓ Done
          </button>
        )}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do ({tasks.todo.length})</h3>
        <div className="task-list">
          {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
        </div>
      </div>
      
      <div className="kanban-column">
        <h3>In Progress ({tasks['in-progress'].length})</h3>
        <div className="task-list">
          {tasks['in-progress'].map(task => <TaskCard key={task._id} task={task} />)}
        </div>
      </div>
      
      <div className="kanban-column">
        <h3>Done ({tasks.done.length})</h3>
        <div className="task-list">
          {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
        </div>
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### React
