---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task management, and intelligent ticket routing
triggers:
  - "help me set up an enterprise user management system"
  - "how do I implement AI analytics for user management"
  - "integrate AI-powered ticket classification"
  - "build a task management system with burnout detection"
  - "create an admin dashboard with user analytics"
  - "implement JWT authentication for user management"
  - "set up AI-based anomaly detection for users"
  - "configure ML service for predictive insights"
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with AI-powered insights. It provides admin controls, user dashboards, Kanban boards, ticket management, and ML-based features like risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Stack:** React.js frontend, Node.js/Express backend, FastAPI ML service, MongoDB database, JWT authentication

**Key AI Features:**
- Automated ticket classification and routing
- Risk prediction based on user behavior
- Anomaly detection for security
- Burnout detection via workload analysis
- Predictive project delay insights

## Installation

### Prerequisites
```bash
# Required
node >= 14.x
python >= 3.8
mongodb >= 4.x
npm >= 6.x
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-management
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-management
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

## Backend API Structure

### Authentication Endpoints

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = await User.create({
      name,
      email,
      password,
      role: role || 'user'
    });

    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );

    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
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

    const isMatch = await user.matchPassword(password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );

    res.json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
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

### Authentication Middleware

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

### User Management Endpoints

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const User = require('../models/User');

// Get all users (Admin only)
router.get('/', protect, authorize('admin'), async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({
      success: true,
      count: users.length,
      data: users
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get single user
router.get('/:id', protect, async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ message: error.message });
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
      return res.status(404).json({ message: 'User not found' });
    }

    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Delete user
router.delete('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json({ success: true, data: {} });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Task Management Endpoints

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const Task = require('../models/Task');

// Create task
router.post('/', protect, authorize('admin', 'manager'), async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user tasks
router.get('/my-tasks', protect, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort('-createdAt');
    
    res.json({ success: true, data: tasks });
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

    // Check if user is assigned to task or is admin
    if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Not authorized' });
    }

    task.status = status;
    if (status === 'done') {
      task.completedAt = Date.now();
    }
    await task.save();

    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Track time
router.post('/:id/track-time', protect, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    task.timeSpent = (task.timeSpent || 0) + timeSpent;
    await task.save();

    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Ticket Management Endpoints

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const { protect } = require('../middleware/auth');
const Ticket = require('../models/Ticket');
const axios = require('axios');

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, priority } = req.body;

    // Call ML service for ticket classification
    let category = 'general';
    let suggestedPriority = priority;

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
      createdBy: req.user.id,
      status: 'open'
    });

    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', protect, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .populate('createdBy', 'name email')
      .sort('-createdAt');
    
    res.json({ success: true, data: tickets });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update ticket status (Admin)
router.patch('/:id', protect, async (req, res) => {
  try {
    const ticket = await Ticket.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    );

    if (!ticket) {
      return res.status(404).json({ message: 'Ticket not found' });
    }

    res.json({ success: true, data: ticket });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: int
    failed_logins: int
    tasks_completed: int
    tasks_overdue: int
    avg_task_completion_time: float

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_working_hours: float
    missed_deadlines: int
    stress_indicators: List[float]

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    login_time: str
    location: str
    device_type: str
    actions_per_session: int

@app.get("/")
def read_root():
    return {
        "service": "Enterprise User Management ML Service",
        "status": "running",
        "endpoints": [
            "/classify-ticket",
            "/predict-risk",
            "/detect-burnout",
            "/detect-anomaly",
            "/predict-project-delay"
        ]
    }

@app.post("/classify-ticket")
def classify_ticket(request: TicketClassificationRequest):
    """
    Classify ticket into categories and suggest priority
    """
    text = f"{request.title} {request.description}".lower()
    
    # Simple rule-based classification (replace with trained model)
    if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
        category = 'technical'
        priority = 'high'
    elif any(word in text for word in ['password', 'access', 'login', 'permission']):
        category = 'access'
        priority = 'high'
    elif any(word in text for word in ['feature', 'request', 'enhancement']):
        category = 'feature_request'
        priority = 'medium'
    elif any(word in text for word in ['question', 'help', 'how to']):
        category = 'support'
        priority = 'low'
    else:
        category = 'general'
        priority = 'medium'
    
    return {
        "category": category,
        "priority": priority,
        "confidence": 0.85
    }

@app.post("/predict-risk")
def predict_risk(request: RiskPredictionRequest):
    """
    Predict user risk level based on behavior patterns
    """
    # Calculate risk score
    risk_score = 0
    
    # Failed logins indicator
    if request.failed_logins > 5:
        risk_score += 30
    elif request.failed_logins > 2:
        risk_score += 15
    
    # Task completion rate
    if request.tasks_completed > 0:
        completion_rate = request.tasks_completed / (request.tasks_completed + request.tasks_overdue)
        if completion_rate < 0.5:
            risk_score += 25
    else:
        risk_score += 20
    
    # Task completion time
    if request.avg_task_completion_time > 48:  # hours
        risk_score += 20
    
    # Login frequency (too low might indicate disengagement)
    if request.login_frequency < 3:  # per week
        risk_score += 15
    
    # Determine risk level
    if risk_score >= 60:
        risk_level = "high"
    elif risk_score >= 30:
        risk_level = "medium"
    else:
        risk_level = "low"
    
    return {
        "user_id": request.user_id,
        "risk_level": risk_level,
        "risk_score": min(risk_score, 100),
        "factors": {
            "failed_logins": request.failed_logins,
            "task_completion_rate": request.tasks_completed / max(request.tasks_completed + request.tasks_overdue, 1),
            "avg_completion_time": request.avg_task_completion_time
        },
        "recommendations": get_risk_recommendations(risk_level)
    }

def get_risk_recommendations(risk_level: str) -> List[str]:
    recommendations = {
        "high": [
            "Review user account for suspicious activity",
            "Consider password reset requirement",
            "Schedule manager check-in",
            "Review task assignments and workload"
        ],
        "medium": [
            "Monitor user activity closely",
            "Provide additional support if needed",
            "Review task deadlines"
        ],
        "low": [
            "Continue regular monitoring",
            "No immediate action required"
        ]
    }
    return recommendations.get(risk_level, [])

@app.post("/detect-burnout")
def detect_burnout(request: BurnoutDetectionRequest):
    """
    Detect potential employee burnout based on workload and behavior
    """
    burnout_score = 0
    
    # Task overload
    if request.tasks_assigned > 20:
        burnout_score += 25
    elif request.tasks_assigned > 15:
        burnout_score += 15
    
    # Completion rate
    completion_rate = request.tasks_completed / max(request.tasks_assigned, 1)
    if completion_rate < 0.6:
        burnout_score += 20
    
    # Working hours
    if request.avg_working_hours > 10:
        burnout_score += 30
    elif request.avg_working_hours > 8:
        burnout_score += 15
    
    # Missed deadlines
    if request.missed_deadlines > 5:
        burnout_score += 25
    elif request.missed_deadlines > 2:
        burnout_score += 15
    
    # Determine burnout level
    if burnout_score >= 60:
        burnout_level = "high"
    elif burnout_score >= 35:
        burnout_level = "moderate"
    else:
        burnout_level = "low"
    
    return {
        "user_id": request.user_id,
        "burnout_level": burnout_level,
        "burnout_score": min(burnout_score, 100),
        "indicators": {
            "task_overload": request.tasks_assigned > 15,
            "excessive_hours": request.avg_working_hours > 8,
            "missed_deadlines": request.missed_deadlines,
            "completion_rate": completion_rate
        },
        "recommendations": get_burnout_recommendations(burnout_level)
    }

def get_burnout_recommendations(level: str) -> List[str]:
    recommendations = {
        "high": [
            "Immediate manager intervention required",
            "Reduce workload significantly",
            "Schedule mandatory time off",
            "Provide mental health resources"
        ],
        "moderate": [
            "Redistribute tasks to balance workload",
            "Monitor working hours",
            "Schedule one-on-one check-in",
            "Encourage breaks and time off"
        ],
        "low": [
            "Continue healthy work patterns",
            "Regular check-ins recommended"
        ]
    }
    return recommendations.get(level, [])

@app.post("/detect-anomaly")
def detect_anomaly(request: AnomalyDetectionRequest):
    """
    Detect anomalous user behavior for security purposes
    """
    anomaly_score = 0
    anomalies = []
    
    # Check login time
    try:
        login_hour = int(request.login_time.split(':')[0])
        if login_hour < 6 or login_hour > 22:
            anomaly_score += 20
            anomalies.append("Unusual login time")
    except:
        pass
    
    # Excessive actions per session
    if request.actions_per_session > 100:
        anomaly_score += 30
        anomalies.append("Unusually high activity")
    
    # Determine if anomalous
    is_anomalous = anomaly_score >= 30
    
    return {
        "user_id": request.user_id,
        "is_anomalous": is_anomalous,
        "anomaly_score": anomaly_score,
        "detected_anomalies": anomalies,
        "severity": "high" if anomaly_score >= 50 else "medium" if anomaly_score >= 30 else "low",
        "action_required": is_anomalous
    }

@app.post("/predict-project-delay")
def predict_project_delay(data: dict):
    """
    Predict if a project is likely to be delayed
    """
    tasks_total = data.get('tasks_total', 0)
    tasks_completed = data.get('tasks_completed', 0)
    days_remaining = data.get('days_remaining', 0)
    avg_completion_time = data.get('avg_completion_time', 0)
    
    if tasks_total == 0:
        return {"delay_probability": 0, "status": "no_tasks"}
    
    tasks_remaining = tasks_total - tasks_completed
    estimated_time_needed = tasks_remaining * avg_completion_time
    
    # Calculate delay probability
    if estimated_time_needed > days_remaining * 24:
        delay_probability = min(0.9, estimated_time_needed / (days_remaining * 24))
    else:
        delay_probability = 0.1
    
    return {
        "delay_probability": round(delay_probability, 2),
        "tasks_remaining": tasks_remaining,
        "estimated_days_needed": round(estimated_time_needed / 24, 1),
        "days_available": days_remaining,
        "status": "at_risk" if delay_probability > 0.6 else "on_track",
        "recommendations": [
            "Reassign tasks to available team members",
            "Extend deadline if possible",
            "Reduce scope of remaining tasks"
        ] if delay_probability > 0.6 else []
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Frontend Integration

### API Service Setup

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
  login: (credentials) => api.post('/api/auth/login', credentials),
  register: (userData) => api.post('/api/auth/register', userData),
  getProfile: () => api.get('/api/auth/profile')
};

// User API
export const userAPI = {
  getAllUsers: () => api.get('/api/users'),
  getUser: (id) => api.get(`/api/users/${id}`),
  updateUser: (id, data) => api.put(`/api/users/${id}`, data),
  deleteUser: (id) => api.delete(`/api/users/${id}`)
};

// Task API
export const taskAPI = {
  createTask: (taskData) => api.post('/api/tasks', taskData),
  getMyTasks: () => api.get('/api/tasks/my-tasks'),
  getAllTasks: () => api.get('/api/tasks'),
  updateTaskStatus: (id, status) => api.patch(`/api/tasks/${id}/status`, { status }),
  trackTime: (id, timeSpent) => api.post(`/api/tasks/${id}/track-time`, { timeSpent })
};

// Ticket API
export const ticketAPI = {
  createTicket: (ticketData) => api.post('/api/tickets', ticketData),
  getMyTickets: () => api.get('/api/tickets/my-tickets'),
  getAllTickets: () => api.get('/api/tickets'),
  updateTicket: (id, data) => api.patch(`/api/tickets/${id}`, data)
};

// ML API
export const mlAPI = {
  classifyTicket: (title, description) => 
    axios.post(`${ML_API_URL}/classify-ticket`, { title, description }),
  
  predictRisk: (userData) => 
    axios.post(`${ML_API_URL}/predict-risk`, userData),
  
  detectBurnout: (userData) => 
    axios.post(`${ML_API_URL}/detect-burnout`, userData),
  
  detectAnomaly: (activityData) => 
    axios.post(`${ML_API_URL}/detect-anomaly`, activityData),
  
  predictProjectDelay: (projectData) => 
    axios.post(`${ML_API_URL}/predict-project-delay`, projectData)
};

export default api;
```

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI, mlAPI } from '../services/api';
import './UserDashboard.css';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    try {
      const tasksResponse = await taskAPI.getMyTasks();
      setTasks(tasksResponse.data.data);

      // Get burnout analysis
      const taskStats = calculateTaskStats(tasksResponse.data.data);
      const burnoutResponse = await mlAPI.detectBurnout(taskStats);
      setBurnoutData(burnoutResponse.data);

      setLoading(false);
    } catch (error) {
      console.error('Error fetching user data:', error);
      setLoading(false);
    }
  };

  const calculateTaskStats = (tasks) => {
    const totalTasks = tasks.length;
    const completedTasks = tasks.filter(t => t.status === 'done').length;
    const overdueTasks = tasks.filter(t => 
      new Date(t.deadline) < new Date() && t.status !== 'done'
    ).length;

    return {
      user_id: localStorage.getItem('userId'),
      tasks_assigned: totalTasks,
      tasks_completed: completedTasks,
      avg_working_hours: 8, // Could be tracked from time logs
      missed_deadlines: overdueTasks,
      stress_indicators: [overdueTasks / totalTasks]
    };
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskAPI.updateTaskStatus(taskId, newStatus);
      fetchUserData(); // Refresh data
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const getTasksByStatus = (status) => {
    return tasks.filter(task => task.status === status);
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="user-dashboard">
      <h1>My Dashboard</h1>

      {/* Burnout Alert */}
      {burnoutData && burnoutData.burnout_level !== 'low' && (
        <div className={`alert alert-${burnoutData.burnout_level}`}>
          <h3>⚠️ Burnout Warning: {burnoutData.burnout_level}</h3>
          <p>Burnout Score: {burnoutData.burnout_score}/100</p>
          <ul>
            {burnoutData.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}

      {/* Task Statistics */}
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Tasks</h3>
          <p className="stat-number">{tasks.length}</p>
        </div>
        <div className="stat-card">
          <h3>To Do</h3>
          <p className="stat-number">{getTasksByStatus('todo').length}</p>
        </div>
        <div className="stat-card">
          <h3>In Progress</h3>
          <p className="stat-number">{getTasksByStatus('in_progress').length}</p>
        </div>
        <div className="stat-card">
          <h3>Completed</h3>
          <p className="stat-number">{getTasksByStatus('done').length}</p>
        </div>
      </div>

      {/* Kanban
