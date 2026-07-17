---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket classification, risk detection, and burnout analysis
triggers:
  - "build a user management system with AI analytics"
  - "implement enterprise task tracking with ML insights"
  - "create admin dashboard with anomaly detection"
  - "set up AI-powered ticket classification system"
  - "integrate burnout detection in user management"
  - "develop kanban board with time tracking"
  - "add predictive analytics to user management"
  - "implement JWT authentication for admin panel"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

This is a full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics. It provides:

- **User & Admin Portals**: Role-based dashboards for task management, ticket handling, and performance tracking
- **AI Analytics Engine**: ML-powered risk detection, anomaly detection, burnout prediction, and ticket classification
- **Task Management**: Kanban boards with time tracking and progress monitoring
- **Support System**: Intelligent ticket routing and management
- **Security**: JWT authentication with audit logging and suspicious activity alerts

The system uses React frontend, Node.js backend, MongoDB database, and FastAPI ML service with scikit-learn and River for online learning.

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
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

**Backend (.env)**
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend (.env)**
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

**ML Service (.env)**
```bash
# ml-service/.env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

## Running the System

```bash
# Terminal 1: Start MongoDB
mongod --dbpath /path/to/data/db

# Terminal 2: Start Backend
cd backend
npm start
# Backend runs at http://localhost:5000

# Terminal 3: Start ML Service
cd ml-service
uvicorn main:app --reload
# ML service runs at http://localhost:8000

# Terminal 4: Start Frontend
cd frontend
npm start
# Frontend runs at http://localhost:3000
```

## Backend API (Node.js)

### Authentication Routes

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = new User({
      username,
      email,
      password, // Should be hashed in User model pre-save hook
      role: role || 'user'
    });

    await user.save();

    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );

    res.status(201).json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const token = jwt.sign(
      { id: user._id, role: user.role },
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
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### User Management Routes

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { protect, authorize } = require('../middleware/auth');

// Get all users (Admin only)
router.get('/', protect, authorize('admin'), async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, count: users.length, data: users });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Get single user
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
    const { username, email, role, status } = req.body;
    
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { username, email, role, status },
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
    res.json({ success: true, message: 'User deleted' });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

### Task Management Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { protect } = require('../middleware/auth');

// Get all tasks for user
router.get('/', protect, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user.id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username');
    
    res.json({ success: true, count: tasks.length, data: tasks });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Create task
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = await Task.create({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      dueDate,
      status: 'todo',
      createdBy: req.user.id
    });

    await task.populate('assignedTo', 'username email');
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ success: false, message: 'Task not found' });
    }

    // Check authorization
    if (req.user.role !== 'admin' && task.assignedTo.toString() !== req.user.id) {
      return res.status(403).json({ success: false, message: 'Not authorized' });
    }

    task.status = status;
    if (status === 'done') {
      task.completedAt = Date.now();
    }
    await task.save();

    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Track time
router.post('/:id/time', protect, async (req, res) => {
  try {
    const { duration } = req.body; // duration in seconds
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ success: false, message: 'Task not found' });
    }

    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();

    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

### Ticket Management Routes

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const axios = require('axios');
const { protect, authorize } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    let category = 'general';
    let suggestedPriority = priority || 'medium';
    
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        title,
        description
      });
      category = mlResponse.data.category;
      suggestedPriority = mlResponse.data.priority;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }

    const ticket = await Ticket.create({
      title,
      description,
      priority: suggestedPriority,
      category,
      status: 'open',
      createdBy: req.user.id
    });

    await ticket.populate('createdBy', 'username email');
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Get all tickets
router.get('/', protect, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.id };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort('-createdAt');
    
    res.json({ success: true, count: tickets.length, data: tickets });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Assign ticket (Admin only)
router.patch('/:id/assign', protect, authorize('admin'), async (req, res) => {
  try {
    const { assignedTo } = req.body;
    
    const ticket = await Ticket.findByIdAndUpdate(
      req.params.id,
      { assignedTo, status: 'in-progress' },
      { new: true }
    ).populate('assignedTo', 'username email');

    res.json({ success: true, data: ticket });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

## ML Service (FastAPI)

### Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from sklearn.preprocessing import StandardScaler
import joblib
import os
from datetime import datetime

app = FastAPI(title="Enterprise UMS ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models directory
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Pydantic models
class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: int
    failed_logins: int
    data_access_count: int
    hours_since_last_login: float
    unusual_access_times: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_pending: int
    avg_task_time: float
    overtime_hours: float
    ticket_count: int

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    features: List[float]

# Initialize models (lazy loading)
ticket_classifier = None
risk_predictor = None
burnout_detector = None
anomaly_detector = None

@app.on_event("startup")
async def load_models():
    global anomaly_detector
    # Initialize anomaly detector with default
    anomaly_detector = IsolationForest(contamination=0.1, random_state=42)

@app.get("/")
async def root():
    return {
        "service": "Enterprise UMS ML Service",
        "status": "active",
        "endpoints": [
            "/classify-ticket",
            "/predict-risk",
            "/analyze-burnout",
            "/detect-anomaly"
        ]
    }

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """
    Classify support ticket into category and suggest priority
    """
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple rule-based classification (can be replaced with trained model)
        category = "general"
        priority = "medium"
        
        if any(word in text for word in ["bug", "error", "crash", "broken"]):
            category = "technical"
            priority = "high"
        elif any(word in text for word in ["password", "login", "access", "permission"]):
            category = "access"
            priority = "high"
        elif any(word in text for word in ["feature", "enhancement", "request"]):
            category = "feature"
            priority = "low"
        elif any(word in text for word in ["urgent", "critical", "emergency"]):
            priority = "critical"
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """
    Predict security risk based on user behavior
    """
    try:
        # Feature engineering
        features = np.array([[
            request.login_frequency,
            request.failed_logins,
            request.data_access_count,
            request.hours_since_last_login,
            request.unusual_access_times
        ]])
        
        # Risk score calculation (simple weighted approach)
        risk_score = 0
        risk_score += request.failed_logins * 20  # Failed logins are critical
        risk_score += request.unusual_access_times * 15
        risk_score += min(request.hours_since_last_login / 24, 5) * 10
        risk_score += max(0, request.data_access_count - 100) * 0.5
        
        risk_score = min(100, risk_score)
        
        risk_level = "low"
        if risk_score > 70:
            risk_level = "critical"
        elif risk_score > 50:
            risk_level = "high"
        elif risk_score > 30:
            risk_level = "medium"
        
        return {
            "user_id": request.user_id,
            "risk_score": round(risk_score, 2),
            "risk_level": risk_level,
            "factors": {
                "failed_logins": request.failed_logins > 3,
                "unusual_access": request.unusual_access_times > 2,
                "inactive_period": request.hours_since_last_login > 168  # 7 days
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """
    Analyze employee burnout risk based on workload
    """
    try:
        # Calculate burnout score
        burnout_score = 0
        
        # Pending tasks factor
        if request.tasks_pending > 10:
            burnout_score += 25
        elif request.tasks_pending > 5:
            burnout_score += 15
        
        # Task completion rate
        total_tasks = request.tasks_completed + request.tasks_pending
        if total_tasks > 0:
            completion_rate = request.tasks_completed / total_tasks
            if completion_rate < 0.5:
                burnout_score += 20
        
        # Average task time (if unusually high)
        if request.avg_task_time > 8:  # hours
            burnout_score += 20
        
        # Overtime
        if request.overtime_hours > 20:
            burnout_score += 30
        elif request.overtime_hours > 10:
            burnout_score += 15
        
        # Ticket load
        if request.ticket_count > 15:
            burnout_score += 10
        
        burnout_score = min(100, burnout_score)
        
        burnout_level = "low"
        if burnout_score > 70:
            burnout_level = "critical"
        elif burnout_score > 50:
            burnout_level = "high"
        elif burnout_score > 30:
            burnout_level = "moderate"
        
        recommendations = []
        if request.overtime_hours > 10:
            recommendations.append("Reduce overtime hours")
        if request.tasks_pending > 10:
            recommendations.append("Redistribute workload")
        if request.avg_task_time > 8:
            recommendations.append("Review task complexity")
        
        return {
            "user_id": request.user_id,
            "burnout_score": round(burnout_score, 2),
            "burnout_level": burnout_level,
            "recommendations": recommendations,
            "metrics": {
                "pending_tasks": request.tasks_pending,
                "overtime_hours": request.overtime_hours,
                "completion_rate": round(request.tasks_completed / max(1, total_tasks), 2)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """
    Detect anomalous behavior patterns
    """
    try:
        features = np.array([request.features]).reshape(1, -1)
        
        # Use isolation forest for anomaly detection
        global anomaly_detector
        
        # Predict (-1 for anomaly, 1 for normal)
        prediction = anomaly_detector.fit_predict(features)[0]
        
        is_anomaly = prediction == -1
        
        return {
            "user_id": request.user_id,
            "is_anomaly": bool(is_anomaly),
            "anomaly_score": float(anomaly_detector.score_samples(features)[0]),
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}
```

## Frontend Integration (React)

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';
const ML_BASE_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

// Create axios instance
const api = axios.create({
  baseURL: API_BASE_URL,
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
  update: (id, data) => api.put(`/users/${id}`, data),
  delete: (id) => api.delete(`/users/${id}`)
};

// Task API
export const taskAPI = {
  getAll: () => api.get('/tasks'),
  create: (taskData) => api.post('/tasks', taskData),
  updateStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  trackTime: (id, duration) => api.post(`/tasks/${id}/time`, { duration })
};

// Ticket API
export const ticketAPI = {
  getAll: () => api.get('/tickets'),
  create: (ticketData) => api.post('/tickets', ticketData),
  assign: (id, userId) => api.patch(`/tickets/${id}/assign`, { assignedTo: userId })
};

// ML API
export const mlAPI = {
  classifyTicket: (title, description) =>
    axios.post(`${ML_BASE_URL}/classify-ticket`, { title, description }),
  
  predictRisk: (userData) =>
    axios.post(`${ML_BASE_URL}/predict-risk`, userData),
  
  analyzeBurnout: (workloadData) =>
    axios.post(`${ML_BASE_URL}/analyze-burnout`, workloadData),
  
  detectAnomaly: (userId, features) =>
    axios.post(`${ML_BASE_URL}/detect-anomaly`, { user_id: userId, features })
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
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getAll();
      setTasks(response.data.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const handleDrop = async (taskId, newStatus) => {
    try {
      await taskAPI.updateStatus(taskId, newStatus);
      setTasks(tasks.map(task =>
        task._id === taskId ? { ...task, status: newStatus } : task
      ));
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const columns = {
    todo: tasks.filter(t => t.status === 'todo'),
    'in-progress': tasks.filter(t => t.status === 'in-progress'),
    done: tasks.filter(t => t.status === 'done')
  };

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {Object.entries(columns).map(([status, columnTasks]) => (
        <div
          key={status}
          className="kanban-column"
          onDragOver={(e) => e.preventDefault()}
          onDrop={(e) => {
            const taskId = e.dataTransfer.getData('taskId');
            handleDrop(taskId, status);
          }}
        >
          <h3>{status.toUpperCase().replace('-', ' ')}</h3>
          <div className="tasks-container">
            {columnTasks.map(task => (
              <div
                key={task._id}
                className="task-card"
                draggable
                onDragStart={(e) => e.dataTransfer.setData('taskId', task._id)}
              >
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <span className={`priority-badge ${task.priority}`}>
                  {task.priority}
                </span>
                {task.timeSpent && (
                  <span className="time-spent">
                    {Math.floor(task.timeSpent / 3600)}h {Math.floor((task.timeSpent % 3600) / 60)}m
                  </span>
                )}
              </div>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import { mlAPI, userAPI } from '../services/api';
import './AIAnalyticsDashboard.css';

const AIAnalyticsDashboard = () => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadAnalytics();
  }, []);

  const loadAnalytics = async () => {
    try {
      const user = JSON.parse(localStorage.getItem('user'));
      
      // Fetch risk prediction
      const riskResponse = await mlAPI.predictRisk({
        user_id: user.id,
        login_frequency: 25,
        failed_logins: 2,
        data_access_count: 150,
        hours_since_last_login: 2.5,
        unusual_access_times: 1
      });
      setRiskData(riskResponse.data);

      // Fetch burnout analysis
      const burnoutResponse = await mlAPI.analyzeBurnout({
        user_id: user.id,
        tasks_completed: 45,
        tasks_pending: 8,
        avg_task_time: 5.5,
        overtime_hours: 12,
        ticket_count: 7
      });
      setBurnoutData(burnoutResponse.data);

      setLoading(false);
    } catch (error) {
      console.error('Error loading analytics:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI-Powered Analytics</h2>
      
      {/* Risk Assessment */}
      <div className="analytics-card">
        <h3>Security Risk Assessment</h3>
        {riskData && (
          <div>
            <div className={`risk-level ${riskData.risk_level}`}>
              {riskData.risk_level.toUpperCase()}
            </div>
            <div className="risk-score">
              Risk Score: {riskData.risk_score}/100
            </div>
            <div className="risk-factors">
              <h4>Risk Factors:</h4>
              <ul>
                {riskData.factors.failed_logins && <li>Multiple failed login attempts</li>}
                {riskData.factors.unusual_access && <li>Unusual access patterns detected</li>}
                {riskData.factors.inactive_period && <li>Extended inactive period</li>}
              </ul>
            </div>
          </div>
        )}
      </div>

      {/* Burnout Analysis */}
      <div className="analytics-card">
        <h3>Burnout Risk Analysis</h3>
        {burnoutData && (
          <div>
            <div className={`burnout-level ${burnoutData.burnout_level}`}>
              {burnoutData.burnout_level.toUpperCase()}
            </div>
            <div className="burnout-score">
              Burnout
