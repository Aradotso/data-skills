---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "create user management dashboard with AI analytics"
  - "implement task tracking with kanban board"
  - "add AI-powered ticket classification"
  - "build user management system with JWT auth"
  - "integrate AI analytics for user behavior"
  - "deploy enterprise management system"
  - "configure user roles and permissions"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System, a full-stack application that combines user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Role-based access control (Admin/User) with JWT authentication
- **Task Management**: Kanban board with time tracking and progress monitoring
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Dashboards**: Separate admin and user interfaces with real-time insights

**Architecture**: React frontend + Node.js/Express backend + FastAPI ML service + MongoDB

## Installation & Setup

### Prerequisites

```bash
# Required
node >= 14.x
python >= 3.8
mongodb >= 4.x
npm or yarn
```

### Clone and Install

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
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
# Runs on http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
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
REACT_APP_ML_SERVICE_URL=http://localhost:8000
EOF

# Start frontend
npm start
# Runs on http://localhost:3000
```

## Key API Endpoints

### Authentication APIs

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// POST /api/auth/register - Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate JWT
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ success: true, token, user });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
});

// POST /api/auth/login - Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      return res.status(401).json({ success: false, message: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ success: false, message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ success: true, token, user });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

### User Management APIs

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { auth, adminAuth } = require('../middleware/auth');

// GET /api/users - Get all users (Admin only)
router.get('/', auth, adminAuth, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, data: users });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// GET /api/users/:id - Get single user
router.get('/:id', auth, async (req, res) => {
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

// PUT /api/users/:id - Update user (Admin only)
router.put('/:id', auth, adminAuth, async (req, res) => {
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
    res.status(400).json({ success: false, message: error.message });
  }
});

// DELETE /api/users/:id - Delete user (Admin only)
router.delete('/:id', auth, adminAuth, async (req, res) => {
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

### Task Management APIs

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { auth } = require('../middleware/auth');

// GET /api/tasks - Get tasks for logged-in user
router.get('/', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    res.json({ success: true, data: tasks });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// POST /api/tasks - Create new task
router.post('/', auth, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
});

// PUT /api/tasks/:id - Update task status
router.put('/:id', auth, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ success: false, message: 'Task not found' });
    }
    
    // Update allowed fields
    const allowedUpdates = ['status', 'priority', 'description', 'timeSpent'];
    Object.keys(req.body).forEach(key => {
      if (allowedUpdates.includes(key)) {
        task[key] = req.body[key];
      }
    });
    
    await task.save();
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

### Support Ticket APIs

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const { auth } = require('../middleware/auth');
const axios = require('axios');

// POST /api/tickets - Create ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for ticket classification
    let category = 'general';
    let suggestedPriority = priority;
    
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      category = mlResponse.data.category;
      suggestedPriority = mlResponse.data.priority;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      priority: suggestedPriority,
      category,
      createdBy: req.user.id,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
});

// GET /api/tickets - Get user tickets
router.get('/', auth, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.id };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, data: tickets });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// PUT /api/tickets/:id - Update ticket status
router.put('/:id', auth, async (req, res) => {
  try {
    const ticket = await Ticket.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status },
      { new: true, runValidators: true }
    );
    
    if (!ticket) {
      return res.status(404).json({ success: false, message: 'Ticket not found' });
    }
    
    res.json({ success: true, data: ticket });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

## AI/ML Service Implementation

### FastAPI ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Dict
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from river import anomaly, drift
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Models storage
MODEL_PATH = os.getenv('MODEL_PATH', './models')

# Request/Response models
class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    priority: str
    confidence: float

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_attempts: int
    failed_logins: int
    unusual_hours: int
    data_access_volume: int

class RiskPredictionResponse(BaseModel):
    risk_level: str
    risk_score: float
    factors: List[str]

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_task_duration: float
    overtime_hours: float
    days_without_break: int

class BurnoutAnalysisResponse(BaseModel):
    burnout_risk: str
    risk_score: float
    recommendations: List[str]

# Initialize ticket classifier
ticket_categories = ['technical', 'billing', 'general', 'hr', 'access']
ticket_priorities = ['low', 'medium', 'high', 'critical']

@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    """AI-based ticket classification and priority assignment"""
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple rule-based classification (replace with trained model)
        category = 'general'
        priority = 'medium'
        confidence = 0.75
        
        # Category detection
        if any(word in text for word in ['password', 'login', 'access', 'permission']):
            category = 'access'
            priority = 'high'
            confidence = 0.85
        elif any(word in text for word in ['bug', 'error', 'crash', 'not working']):
            category = 'technical'
            priority = 'high'
            confidence = 0.90
        elif any(word in text for word in ['invoice', 'payment', 'billing', 'refund']):
            category = 'billing'
            priority = 'medium'
            confidence = 0.80
        elif any(word in text for word in ['leave', 'salary', 'hr', 'policy']):
            category = 'hr'
            priority = 'low'
            confidence = 0.70
        
        # Priority escalation
        if any(word in text for word in ['urgent', 'critical', 'asap', 'emergency']):
            priority = 'critical'
            confidence = 0.95
        
        return TicketClassificationResponse(
            category=category,
            priority=priority,
            confidence=confidence
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    """Predict user security risk based on behavior patterns"""
    try:
        # Feature engineering
        failed_login_rate = request.failed_logins / max(request.login_attempts, 1)
        
        # Calculate risk score (0-100)
        risk_score = 0.0
        factors = []
        
        if failed_login_rate > 0.3:
            risk_score += 30
            factors.append("High failed login rate")
        
        if request.unusual_hours > 5:
            risk_score += 25
            factors.append("Frequent unusual hour access")
        
        if request.data_access_volume > 1000:
            risk_score += 25
            factors.append("Abnormal data access volume")
        
        if request.login_attempts > 20:
            risk_score += 20
            factors.append("Excessive login attempts")
        
        # Determine risk level
        if risk_score >= 70:
            risk_level = "critical"
        elif risk_score >= 50:
            risk_level = "high"
        elif risk_score >= 30:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return RiskPredictionResponse(
            risk_level=risk_level,
            risk_score=risk_score,
            factors=factors if factors else ["No significant risk factors detected"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout", response_model=BurnoutAnalysisResponse)
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze employee burnout risk based on workload patterns"""
    try:
        # Calculate completion rate
        completion_rate = request.tasks_completed / max(request.tasks_assigned, 1)
        
        # Burnout risk score
        risk_score = 0.0
        recommendations = []
        
        # Low completion rate
        if completion_rate < 0.5:
            risk_score += 30
            recommendations.append("Review task assignments and reduce workload")
        
        # Long task duration
        if request.avg_task_duration > 8:
            risk_score += 25
            recommendations.append("Break down complex tasks into smaller units")
        
        # Excessive overtime
        if request.overtime_hours > 10:
            risk_score += 25
            recommendations.append("Limit overtime and ensure work-life balance")
        
        # No breaks
        if request.days_without_break > 10:
            risk_score += 20
            recommendations.append("Schedule regular breaks and time off")
        
        # Determine burnout risk
        if risk_score >= 70:
            burnout_risk = "critical"
        elif risk_score >= 50:
            burnout_risk = "high"
        elif risk_score >= 30:
            burnout_risk = "medium"
        else:
            burnout_risk = "low"
            recommendations = ["Employee workload appears healthy"]
        
        return BurnoutAnalysisResponse(
            burnout_risk=burnout_risk,
            risk_score=risk_score,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics Service"}
```

## Frontend Integration Examples

### React Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const res = await axios.get(`${API_URL}/api/auth/me`);
      setUser(res.data.data);
    } catch (error) {
      console.error('Failed to fetch user:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    
    const { token, user } = res.data;
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    
    return user;
  };

  const register = async (userData) => {
    const res = await axios.post(`${API_URL}/api/auth/register`, userData);
    
    const { token, user } = res.data;
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
    delete axios.defaults.headers.common['Authorization'];
  };

  return (
    <AuthContext.Provider value={{
      user,
      token,
      loading,
      login,
      register,
      logout,
      isAuthenticated: !!user,
      isAdmin: user?.role === 'admin'
    }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${API_URL}/api/tasks`);
      const allTasks = res.data.data;
      
      setTasks({
        todo: allTasks.filter(t => t.status === 'todo'),
        inProgress: allTasks.filter(t => t.status === 'in_progress'),
        done: allTasks.filter(t => t.status === 'done')
      });
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${API_URL}/api/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const handleDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const handleDrop = (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');
    
    if (currentStatus !== newStatus) {
      updateTaskStatus(taskId, newStatus);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const renderTaskCard = (task, status) => (
    <div
      key={task._id}
      className="task-card"
      draggable
      onDragStart={(e) => handleDragStart(e, task._id, status)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>
          {task.priority}
        </span>
        <span className="time-spent">
          {Math.floor(task.timeSpent / 60)}h {task.timeSpent % 60}m
        </span>
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <div
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'todo')}
        onDragOver={handleDragOver}
      >
        <h3>To Do ({tasks.todo.length})</h3>
        <div className="task-list">
          {tasks.todo.map(task => renderTaskCard(task, 'todo'))}
        </div>
      </div>

      <div
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'in_progress')}
        onDragOver={handleDragOver}
      >
        <h3>In Progress ({tasks.inProgress.length})</h3>
        <div className="task-list">
          {tasks.inProgress.map(task => renderTaskCard(task, 'in_progress'))}
        </div>
      </div>

      <div
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'done')}
        onDragOver={handleDragOver}
      >
        <h3>Done ({tasks.done.length})</h3>
        <div className="task-list">
          {tasks.done.map(task => renderTaskCard(task, 'done'))}
        </div>
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard Component

```javascript
// frontend/src/components/AIAnalyticsDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { LineChart, Line, BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, Legend } from 'recharts';
import './AIAnalyticsDashboard.css';

const AIAnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    burnoutAlerts: [],
    ticketTrends: [],
    anomalies: []
  });

  const ML_SERVICE_URL = process.env.REACT_APP_ML_SERVICE_URL;
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch user data
      const usersRes = await axios.get(`${API_URL}/api/users`);
      const users = usersRes.data.data;

      // Analyze each user for risk
      const riskAnalysis = await Promise.all(
        users.map(async (user) => {
          try {
            const riskRes = await axios.post(`${ML_SERVICE_URL}/predict-risk`, {
              user_id: user._id,
              login_attempts: user.loginAttempts || 0,
              failed_logins: user.failedLogins || 0,
              unusual_hours: user.unusualHourAccess || 0,
              data_access_volume: user.dataAccessVolume || 0
            });
            
            return {
              ...user,
              riskLevel: riskRes.data.risk_level,
              riskScore: riskRes.data.risk_score,
              riskFactors: riskRes.data.factors
            };
          } catch (error) {
            return { ...user, riskLevel: 'unknown', riskScore: 0 };
          }
        })
      );

      // Filter high-risk users
      const highRiskUsers = riskAnalysis.filter(
        u => u.riskLevel === 'high' || u.riskLevel === 'critical'
      );

      // Analyze burnout
      const burnoutAnalysis = await Promise.all(
        users.map(async (user) => {
          try {
            const burnoutRes = await axios.post(`${ML_SERVICE_URL}/analyze-burnout`, {
              user_id: user._id,
              tasks_assigned: user.tasksAssigned || 0,
              tasks_completed: user.tasksCompleted || 0,
              avg_task_duration: user.avgTaskDuration || 0,
              overtime_hours: user.overtimeHours || 0,
              days_without_break: user.daysWithoutBreak || 0
            });
            
            return {
              ...user,
              burnoutRisk: burnoutRes.data.burnout_risk,
              burnoutScore: burnoutRes.data.risk_score,
              recommendations: burnoutRes.data.recommendations
            };
          } catch (error) {
            return { ...user, burnoutRisk: 'unknown', burnoutScore: 0 };
          }
        })
      );

      const burnoutAlerts = burnoutAnalysis.filter(
        u => u.burnoutRisk === 'high' || u.burnoutRisk === 'critical'
      );

      setAnalytics({
        riskUsers: highRiskUsers,
        burnoutAlerts: burnoutAlerts,
        ticketTrends: [], // Populate from ticket data
        anomalies: []
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics Dashboard</h2>

      <div className="analytics-grid">
        <div className="analytics-card">
          <h3>Security Risk Alerts</h3>
          <div className="risk-users-list">
            {analytics.riskUsers.length === 0 ? (
              <p>No high-risk users detected</p>
            ) : (
              analytics.riskUsers.map(user => (
                <div key={user._id} className="risk-user-item">
                  <div className="user-info">
                    <strong>{user.username}</strong>
                    <span className={`risk-badge ${user.riskLevel}`}>
                      {user.riskLevel}
                    </span>
                  </div>
                  <div className="risk-score">
                    Risk Score: {user.riskScore.toFixed(1)}
                  </div>
                  <ul className="risk-factors">
                    {user.riskFactors.map((factor, idx) => (
                      <li key={idx}>{factor}</li>
                    ))}
                  </ul>
                </div>
              ))
            )}
          </div>
        </div>

        <div className="analytics-card">
          <h3>Burnout Detection</h3>
          <div className="burnout-alerts-list">
            {analytics.burnoutAlerts.length === 0 ? (
              <p>No burnout risks detected</p>
            ) : (
              analytics.burnoutAlerts.map(user => (
                <div key={user._id} className="burnout-alert-item">
                  <div className="user-info">
                    <strong>{user.username}</strong>
                    
