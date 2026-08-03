---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, task management, and organizational insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management dashboard with AI insights"
  - "implement task tracking with burnout detection"
  - "build admin panel with anomaly detection"
  - "add AI-powered ticket classification system"
  - "configure user management with predictive analytics"
  - "integrate machine learning for user behavior analysis"
  - "develop organization management with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript/Python application that combines traditional user management with AI-powered insights. The system provides role-based access control, task management with Kanban boards, support ticket handling, and ML-driven analytics including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Architecture:**
- **Frontend:** React.js with JWT authentication
- **Backend:** Node.js with REST APIs
- **ML Service:** FastAPI with scikit-learn and River (online learning)
- **Database:** MongoDB

## Installation

### Clone and Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install
```

Create `.env` file in backend directory:

```bash
PORT=5000
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=24h
NODE_ENV=development
```

Start backend server:

```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```bash
ML_MODEL_PATH=./models
API_KEY=your_api_key_for_ml_service
MONGO_URI=your_mongodb_connection_string
```

Start ML service:

```bash
uvicorn main:app --reload
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_JWT_SECRET=your_jwt_secret_key
```

Start frontend:

```bash
npm start
# Frontend runs at http://localhost:3000
```

## Core Components

### Authentication System

**Backend - JWT Authentication (backend/middleware/auth.js):**

```javascript
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
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

const roleCheck = (roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Access denied' });
    }
    next();
  };
};

module.exports = { authMiddleware, roleCheck };
```

**Frontend - Login Component (frontend/src/components/Login.js):**

```javascript
import React, { useState } from 'react';
import axios from 'axios';

const Login = () => {
  const [credentials, setCredentials] = useState({ email: '', password: '' });
  const [error, setError] = useState('');

  const handleLogin = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_API_URL}/auth/login`,
        credentials
      );
      
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
      
      // Redirect based on role
      if (response.data.user.role === 'admin') {
        window.location.href = '/admin/dashboard';
      } else {
        window.location.href = '/user/dashboard';
      }
    } catch (err) {
      setError(err.response?.data?.message || 'Login failed');
    }
  };

  return (
    <form onSubmit={handleLogin}>
      <input
        type="email"
        placeholder="Email"
        value={credentials.email}
        onChange={(e) => setCredentials({...credentials, email: e.target.value})}
        required
      />
      <input
        type="password"
        placeholder="Password"
        value={credentials.password}
        onChange={(e) => setCredentials({...credentials, password: e.target.value})}
        required
      />
      {error && <div className="error">{error}</div>}
      <button type="submit">Login</button>
    </form>
  );
};

export default Login;
```

### User Management API

**Backend - User Controller (backend/controllers/userController.js):**

```javascript
const User = require('../models/User');
const bcrypt = require('bcryptjs');

// Get all users (Admin only)
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, data: users });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};

// Create new user
exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }
    
    const hashedPassword = await bcrypt.hash(password, 10);
    
    const user = await User.create({
      name,
      email,
      password: hashedPassword,
      role: role || 'user',
      department
    });
    
    res.status(201).json({ 
      success: true, 
      data: { ...user._doc, password: undefined } 
    });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    if (updates.password) {
      updates.password = await bcrypt.hash(updates.password, 10);
    }
    
    const user = await User.findByIdAndUpdate(
      id, 
      updates, 
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};

// Delete user
exports.deleteUser = async (req, res) => {
  try {
    const { id } = req.params;
    const user = await User.findByIdAndDelete(id);
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json({ success: true, message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};
```

### Task Management System

**Backend - Task Model (backend/models/Task.js):**

```javascript
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    required: true
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
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  dueDate: {
    type: Date,
    required: true
  },
  timeTracked: {
    type: Number,
    default: 0 // in minutes
  },
  tags: [String],
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

**Frontend - Kanban Board (frontend/src/components/KanbanBoard.js):**

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        'in-progress': response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      fetchTasks(); // Refresh board
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
          Due: {new Date(task.dueDate).toLocaleDateString()}
        </span>
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace('-', ' ').toUpperCase()}</h3>
          <div className="task-list">
            {tasks[status].map(task => (
              <TaskCard key={task._id} task={task} />
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Support Ticket System

**Backend - Ticket Controller (backend/controllers/ticketController.js):**

```javascript
const Ticket = require('../models/Ticket');

// Create ticket
exports.createTicket = async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    const ticket = await Ticket.create({
      title,
      description,
      priority,
      createdBy: req.user.id,
      status: 'open'
    });
    
    // Call ML service for classification
    const mlResponse = await fetch(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          title,
          description,
          priority
        })
      }
    );
    
    const classification = await mlResponse.json();
    ticket.category = classification.category;
    ticket.suggestedAssignee = classification.assignee;
    await ticket.save();
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};

// Get user tickets
exports.getUserTickets = async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .populate('assignedTo', 'name email')
      .sort('-createdAt');
    
    res.json({ success: true, data: tickets });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};
```

## AI/ML Features

### ML Service Structure (ml-service/main.py)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Dict
import os
from dotenv import load_dotenv

load_dotenv()

app = FastAPI(title="Enterprise User Management ML Service")

# Load models
MODEL_PATH = os.getenv('ML_MODEL_PATH', './models')

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    priority: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_attempts: int
    failed_logins: int
    access_hours: List[int]
    unusual_locations: int
    data_access_volume: float

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_task_duration: float
    overtime_hours: float
    last_break_hours: float

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """AI-powered ticket classification and routing"""
    try:
        # Simple keyword-based classification (replace with trained model)
        text = f"{request.title} {request.description}".lower()
        
        categories = {
            'technical': ['bug', 'error', 'crash', 'not working', 'failed'],
            'access': ['login', 'password', 'permission', 'access denied'],
            'feature': ['request', 'enhancement', 'new feature', 'add'],
            'general': []
        }
        
        category = 'general'
        for cat, keywords in categories.items():
            if any(keyword in text for keyword in keywords):
                category = cat
                break
        
        # Assign based on category and priority
        assignee_map = {
            'technical': 'tech-support-team',
            'access': 'security-team',
            'feature': 'product-team',
            'general': 'support-team'
        }
        
        return {
            'category': category,
            'assignee': assignee_map.get(category, 'support-team'),
            'confidence': 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict security risk based on user behavior"""
    try:
        # Calculate risk score
        risk_score = 0.0
        
        # Failed login attempts (0-30 points)
        if request.failed_logins > 5:
            risk_score += min(30, request.failed_logins * 3)
        
        # Unusual access hours (0-25 points)
        unusual_hours = sum(1 for h in request.access_hours if h < 6 or h > 22)
        risk_score += unusual_hours * 5
        
        # Unusual locations (0-25 points)
        risk_score += min(25, request.unusual_locations * 8)
        
        # High data access (0-20 points)
        if request.data_access_volume > 1000:
            risk_score += min(20, (request.data_access_volume / 100))
        
        risk_level = 'low'
        if risk_score > 60:
            risk_level = 'high'
        elif risk_score > 30:
            risk_level = 'medium'
        
        return {
            'user_id': request.user_id,
            'risk_score': min(100, risk_score),
            'risk_level': risk_level,
            'recommendations': get_risk_recommendations(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze employee burnout risk"""
    try:
        burnout_score = 0.0
        
        # Task completion rate
        completion_rate = (
            request.tasks_completed / request.tasks_assigned 
            if request.tasks_assigned > 0 else 1.0
        )
        
        if completion_rate < 0.6:
            burnout_score += 25
        
        # Workload (task duration)
        if request.avg_task_duration > 8:  # hours
            burnout_score += 20
        
        # Overtime hours
        if request.overtime_hours > 10:
            burnout_score += min(30, request.overtime_hours * 2)
        
        # Break time
        if request.last_break_hours > 48:
            burnout_score += 25
        
        burnout_level = 'low'
        if burnout_score > 60:
            burnout_level = 'high'
        elif burnout_score > 35:
            burnout_level = 'medium'
        
        return {
            'user_id': request.user_id,
            'burnout_score': min(100, burnout_score),
            'burnout_level': burnout_level,
            'recommendations': get_burnout_recommendations(burnout_level),
            'suggested_actions': [
                'Redistribute tasks' if completion_rate < 0.6 else None,
                'Mandatory break' if request.last_break_hours > 72 else None,
                'Reduce overtime' if request.overtime_hours > 15 else None
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-project-delay")
async def predict_project_delay(data: Dict):
    """Predict if project will be delayed"""
    try:
        tasks_total = data.get('tasks_total', 0)
        tasks_completed = data.get('tasks_completed', 0)
        days_elapsed = data.get('days_elapsed', 0)
        days_remaining = data.get('days_remaining', 0)
        team_size = data.get('team_size', 1)
        
        # Calculate velocity
        velocity = tasks_completed / days_elapsed if days_elapsed > 0 else 0
        required_velocity = (tasks_total - tasks_completed) / days_remaining if days_remaining > 0 else float('inf')
        
        delay_probability = 0.0
        if required_velocity > velocity * 1.5:
            delay_probability = 0.8
        elif required_velocity > velocity:
            delay_probability = 0.5
        else:
            delay_probability = 0.2
        
        return {
            'delay_probability': delay_probability,
            'current_velocity': round(velocity, 2),
            'required_velocity': round(required_velocity, 2),
            'estimated_delay_days': max(0, int((tasks_total - tasks_completed) / velocity - days_remaining)) if velocity > 0 else 0,
            'recommendations': get_project_recommendations(delay_probability)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_risk_recommendations(level: str) -> List[str]:
    recommendations = {
        'low': ['Continue monitoring user activity'],
        'medium': ['Enable 2FA', 'Review recent activities', 'Send security notification'],
        'high': ['Temporary account suspension', 'Immediate security review', 'Force password reset']
    }
    return recommendations.get(level, [])

def get_burnout_recommendations(level: str) -> List[str]:
    recommendations = {
        'low': ['Maintain current workload', 'Regular check-ins'],
        'medium': ['Redistribute some tasks', 'Encourage breaks', 'Monitor closely'],
        'high': ['Immediate workload reduction', 'Mandatory time off', 'HR intervention']
    }
    return recommendations.get(level, [])

def get_project_recommendations(probability: float) -> List[str]:
    if probability > 0.7:
        return ['Increase team size', 'Reduce scope', 'Extend deadline', 'Daily standups']
    elif probability > 0.4:
        return ['Monitor velocity closely', 'Remove blockers', 'Consider additional resources']
    return ['Project on track', 'Maintain current pace']

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

### Frontend - AI Analytics Dashboard (frontend/src/components/AIAnalytics.js)

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    burnoutAlerts: [],
    projectInsights: null
  });

  useEffect(() => {
    fetchAIAnalytics();
  }, []);

  const fetchAIAnalytics = async () => {
    try {
      const token = localStorage.getItem('token');
      
      // Fetch risk analysis
      const riskResponse = await axios.get(
        `${process.env.REACT_APP_API_URL}/analytics/risk-users`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      // Fetch burnout analysis
      const burnoutResponse = await axios.get(
        `${process.env.REACT_APP_API_URL}/analytics/burnout`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      setAnalytics({
        riskUsers: riskResponse.data.data || [],
        burnoutAlerts: burnoutResponse.data.data || [],
        projectInsights: null
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const analyzeBurnout = async (userId) => {
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/analyze-burnout`,
        {
          user_id: userId,
          tasks_assigned: 25,
          tasks_completed: 15,
          avg_task_duration: 6.5,
          overtime_hours: 12,
          last_break_hours: 60
        }
      );
      
      return response.data;
    } catch (error) {
      console.error('Burnout analysis error:', error);
    }
  };

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics Dashboard</h2>
      
      <div className="analytics-section">
        <h3>High Risk Users</h3>
        {analytics.riskUsers.map(user => (
          <div key={user.user_id} className="alert-card risk">
            <h4>{user.name}</h4>
            <p>Risk Score: {user.risk_score}/100</p>
            <p>Level: <span className={user.risk_level}>{user.risk_level}</span></p>
            <ul>
              {user.recommendations?.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        ))}
      </div>
      
      <div className="analytics-section">
        <h3>Burnout Alerts</h3>
        {analytics.burnoutAlerts.map(alert => (
          <div key={alert.user_id} className="alert-card burnout">
            <h4>{alert.name}</h4>
            <p>Burnout Score: {alert.burnout_score}/100</p>
            <p>Level: <span className={alert.burnout_level}>{alert.burnout_level}</span></p>
            <ul>
              {alert.recommendations?.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Common Patterns

### API Request Pattern with Authentication

```javascript
// Reusable API client (frontend/src/utils/api.js)
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// Add token to every request
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle token expiration
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Time Tracking Implementation

```javascript
// Frontend - Time Tracker (frontend/src/components/TimeTracker.js)
import React, { useState, useEffect } from 'react';
import api from '../utils/api';

const TimeTracker = ({ taskId }) => {
  const [isTracking, setIsTracking] = useState(false);
  const [seconds, setSeconds] = useState(0);
  const [intervalId, setIntervalId] = useState(null);

  const startTracking = () => {
    setIsTracking(true);
    const id = setInterval(() => {
      setSeconds(prev => prev + 1);
    }, 1000);
    setIntervalId(id);
  };

  const stopTracking = async () => {
    clearInterval(intervalId);
    setIsTracking(false);
    
    // Save time to backend
    try {
      await api.patch(`/tasks/${taskId}/time`, {
        timeTracked: Math.floor(seconds / 60) // Convert to minutes
      });
      setSeconds(0);
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const mins = Math.floor((secs % 3600) / 60);
    const s = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      {!isTracking ? (
        <button onClick={startTracking}>Start</button>
      ) : (
        <button onClick={stopTracking}>Stop</button>
      )}
    </div>
  );
};

export default TimeTracker;
```

## Configuration

### Backend Routes Setup (backend/routes/index.js)

```javascript
const express = require('express');
const router = express.Router();
const { authMiddleware, roleCheck } = require('../middleware/auth');
const userController = require('../controllers/userController');
const taskController = require('../controllers/taskController');
const ticketController = require('../controllers/ticketController');

// Auth routes
router.post('/auth/login', require('../controllers/authController').login);
router.post('/auth/register', require('../controllers/authController').register);

// User routes
router.get('/users', authMiddleware, roleCheck(['admin']), userController.getAllUsers);
router.post('/users', authMiddleware, roleCheck(['admin']), userController.createUser);
router.patch('/users/:id', authMiddleware, roleCheck(['admin']), userController.updateUser);
router.delete('/users/:id', authMiddleware, roleCheck(['admin']), userController.deleteUser);

// Task routes
router.get('/tasks', authMiddleware, taskController.getUserTasks);
router.post('/tasks', authMiddleware, roleCheck(['admin', 'manager']), taskController.createTask);
router.patch('/tasks/:id', authMiddleware, taskController.updateTask);
router.patch('/tasks/:id/time', authMiddleware, taskController.updateTimeTracked);

// Ticket routes
router.get('/tickets', authMiddleware, ticketController.getUserTickets);
router.post('/tickets', authMiddleware, ticketController.createTicket);
router.patch('/tickets/:id', authMiddleware, ticketController.updateTicket);

// Analytics routes
router.get('/analytics/risk-users', authMiddleware, roleCheck(['admin']), require('../controllers/analyticsController').getRiskUsers);
router.get('/analytics/burnout', authMiddleware, roleCheck(['admin']), require('../controllers/analyticsController').getBurnoutAnalysis);

module.exports = router;
```

## Troubleshooting

### Common Issues

**1. JWT Token Expired**

```javascript
// Check token validity before making requests
const isTokenValid = () => {
  const token = localStorage.getItem('token');
  if (!token) return false;
  
  try {
    const decoded = JSON.parse(atob(token.split('.')[1]));
    return decoded.exp * 1000 > Date.now();
  } catch {
    return false;
  }
};

if (!isTokenValid()) {
  // Redirect to login
  window.location.href = '/login';
}
```

**2. CORS Issues**

```javascript
