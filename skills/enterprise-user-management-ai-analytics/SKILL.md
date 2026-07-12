---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create admin dashboard with user tracking"
  - "integrate AI ticket classification system"
  - "build task management with kanban board"
  - "deploy user management system with ML service"
  - "configure JWT authentication for enterprise app"
  - "implement burnout detection and risk prediction"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack web application that combines user management, task tracking, and support ticket systems with AI-powered insights. It provides risk detection, anomaly detection, burnout analysis, and predictive project analytics to improve organizational efficiency.

**Key Components:**
- **Frontend**: React.js application with admin and user dashboards
- **Backend**: Node.js/Express REST API with MongoDB
- **ML Service**: FastAPI service with scikit-learn and River for online learning
- **Features**: JWT authentication, role-based access, Kanban boards, AI ticket routing

## Installation

### Prerequisites
```bash
# Required tools
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
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
EOF

# Start frontend
npm start
# Runs at http://localhost:3000
```

## Backend API Architecture

### User Authentication

```javascript
// backend/controllers/authController.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register new user
exports.register = async (req, res) => {
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
};

// Login user
exports.login = async (req, res) => {
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
};
```

### Middleware for Authentication

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

### Task Management API

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

// Create task (Admin only)
exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = await Task.create({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: 'todo',
      dueDate,
      createdBy: req.user._id
    });
    
    res.status(201).json({
      success: true,
      task
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Get user tasks
exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user._id })
      .populate('createdBy', 'name email')
      .sort('-createdAt');
    
    res.json({
      success: true,
      count: tasks.length,
      tasks
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Update task status
exports.updateTaskStatus = async (req, res) => {
  try {
    const { status, timeSpent } = req.body;
    
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    if (task.assignedTo.toString() !== req.user._id.toString()) {
      return res.status(403).json({ message: 'Not authorized' });
    }
    
    task.status = status || task.status;
    if (timeSpent) {
      task.timeSpent = (task.timeSpent || 0) + timeSpent;
    }
    
    await task.save();
    
    res.json({
      success: true,
      task
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### Support Ticket System

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

// Create support ticket with AI classification
exports.createTicket = async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    let category = 'general';
    let suggestedPriority = priority;
    
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { text: `${title} ${description}` }
      );
      category = mlResponse.data.category;
      suggestedPriority = mlResponse.data.priority;
    } catch (mlError) {
      console.log('ML classification failed, using defaults');
    }
    
    const ticket = await Ticket.create({
      title,
      description,
      category,
      priority: suggestedPriority,
      createdBy: req.user._id,
      status: 'open'
    });
    
    res.status(201).json({
      success: true,
      ticket
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Get user tickets
exports.getUserTickets = async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user._id })
      .sort('-createdAt');
    
    res.json({
      success: true,
      count: tickets.length,
      tickets
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

## ML Service Implementation

### FastAPI Application Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models (loaded at startup)
models = {}

@app.on_event("startup")
async def load_models():
    """Load ML models on startup"""
    model_path = os.getenv('MODEL_PATH', './models')
    try:
        models['ticket_classifier'] = joblib.load(f'{model_path}/ticket_classifier.pkl')
        models['risk_predictor'] = joblib.load(f'{model_path}/risk_predictor.pkl')
        models['burnout_detector'] = joblib.load(f'{model_path}/burnout_detector.pkl')
    except Exception as e:
        print(f"Warning: Could not load models: {e}")

# Request models
class TicketRequest(BaseModel):
    text: str

class UserBehaviorRequest(BaseModel):
    user_id: str
    login_count: int
    failed_logins: int
    avg_session_duration: float
    unusual_access_times: int
    data_access_volume: int

class BurnoutRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_pending: int
    avg_task_time: float
    overtime_hours: float
    missed_deadlines: int

class ProjectInsightRequest(BaseModel):
    project_id: str
    tasks_total: int
    tasks_completed: int
    avg_completion_rate: float
    team_size: int
    deadline_days: int
```

### AI Ticket Classification

```python
# ml-service/main.py (continued)
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import re

# Initialize vectorizer
vectorizer = TfidfVectorizer(max_features=100, stop_words='english')

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """Classify support ticket and suggest priority"""
    try:
        text = request.text.lower()
        
        # Simple rule-based classification
        categories = {
            'technical': ['error', 'bug', 'crash', 'not working', 'broken'],
            'account': ['login', 'password', 'access', 'permission'],
            'feature': ['request', 'enhancement', 'suggestion', 'improve'],
            'billing': ['payment', 'invoice', 'charge', 'subscription']
        }
        
        category = 'general'
        for cat, keywords in categories.items():
            if any(keyword in text for keyword in keywords):
                category = cat
                break
        
        # Determine priority
        urgent_keywords = ['urgent', 'critical', 'asap', 'emergency', 'down']
        priority = 'high' if any(keyword in text for keyword in urgent_keywords) else 'medium'
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-risk")
async def detect_risk(request: UserBehaviorRequest):
    """Detect risk based on user behavior patterns"""
    try:
        # Calculate risk score
        risk_score = 0
        risk_factors = []
        
        # Failed login attempts
        if request.failed_logins > 3:
            risk_score += 30
            risk_factors.append("Multiple failed login attempts")
        
        # Unusual access patterns
        if request.unusual_access_times > 5:
            risk_score += 25
            risk_factors.append("Unusual access times detected")
        
        # High data access volume
        if request.data_access_volume > 1000:
            risk_score += 20
            risk_factors.append("Unusually high data access")
        
        # Short session durations with high frequency
        if request.login_count > 20 and request.avg_session_duration < 5:
            risk_score += 15
            risk_factors.append("Suspicious session patterns")
        
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
            "risk_factors": risk_factors,
            "recommendation": "Review user activity" if risk_level != "low" else "No action needed"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutRequest):
    """Detect employee burnout based on workload metrics"""
    try:
        burnout_score = 0
        indicators = []
        
        # High pending tasks
        workload_ratio = request.tasks_pending / max(request.tasks_completed, 1)
        if workload_ratio > 2:
            burnout_score += 25
            indicators.append("High pending task ratio")
        
        # Excessive overtime
        if request.overtime_hours > 20:
            burnout_score += 30
            indicators.append("Excessive overtime hours")
        
        # Missed deadlines
        if request.missed_deadlines > 3:
            burnout_score += 25
            indicators.append("Frequent missed deadlines")
        
        # Increasing task completion time
        if request.avg_task_time > 8:
            burnout_score += 20
            indicators.append("Decreasing productivity")
        
        # Determine burnout level
        if burnout_score >= 60:
            level = "high"
            action = "Immediate intervention required"
        elif burnout_score >= 30:
            level = "moderate"
            action = "Monitor closely and redistribute workload"
        else:
            level = "low"
            action = "Continue monitoring"
        
        return {
            "user_id": request.user_id,
            "burnout_level": level,
            "burnout_score": min(burnout_score, 100),
            "indicators": indicators,
            "recommended_action": action
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-project-delay")
async def predict_project_delay(request: ProjectInsightRequest):
    """Predict potential project delays"""
    try:
        delay_probability = 0
        factors = []
        
        # Calculate completion rate
        completion_rate = request.tasks_completed / max(request.tasks_total, 1)
        
        # Low completion rate with tight deadline
        if completion_rate < 0.5 and request.deadline_days < 14:
            delay_probability += 40
            factors.append("Low completion rate with approaching deadline")
        
        # Small team with many tasks
        tasks_per_person = request.tasks_total / max(request.team_size, 1)
        if tasks_per_person > 10:
            delay_probability += 25
            factors.append("High task load per team member")
        
        # Below average completion rate
        if request.avg_completion_rate < 0.6:
            delay_probability += 20
            factors.append("Below average completion rate")
        
        # Critical deadline proximity
        if request.deadline_days < 7 and completion_rate < 0.8:
            delay_probability += 15
            factors.append("Critical deadline approaching")
        
        delay_risk = "high" if delay_probability >= 60 else "medium" if delay_probability >= 30 else "low"
        
        return {
            "project_id": request.project_id,
            "delay_probability": min(delay_probability, 100),
            "delay_risk": delay_risk,
            "contributing_factors": factors,
            "recommendation": "Add resources" if delay_risk == "high" else "Monitor progress"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {"status": "healthy", "models_loaded": len(models)}
```

## Frontend Implementation

### React Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  const API_URL = process.env.REACT_APP_API_URL;
  
  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      loadUser();
    } else {
      setLoading(false);
    }
  }, []);
  
  const loadUser = async () => {
    try {
      const res = await axios.get(`${API_URL}/auth/me`);
      setUser(res.data.user);
    } catch (error) {
      console.error('Load user failed:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };
  
  const login = async (email, password) => {
    try {
      const res = await axios.post(`${API_URL}/auth/login`, {
        email,
        password
      });
      
      localStorage.setItem('token', res.data.token);
      axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
      setUser(res.data.user);
      
      return { success: true };
    } catch (error) {
      return {
        success: false,
        message: error.response?.data?.message || 'Login failed'
      };
    }
  };
  
  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };
  
  return (
    <AuthContext.Provider value={{ user, loading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

export default AuthContext;
```

### Kanban Task Board Component

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
  const [loading, setLoading] = useState(true);
  
  const API_URL = process.env.REACT_APP_API_URL;
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${API_URL}/tasks/user`);
      const allTasks = res.data.tasks;
      
      setTasks({
        todo: allTasks.filter(t => t.status === 'todo'),
        inProgress: allTasks.filter(t => t.status === 'inProgress'),
        done: allTasks.filter(t => t.status === 'done')
      });
    } catch (error) {
      console.error('Fetch tasks failed:', error);
    } finally {
      setLoading(false);
    }
  };
  
  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${API_URL}/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Update task failed:', error);
    }
  };
  
  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };
  
  const handleDrop = (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, newStatus);
  };
  
  const handleDragOver = (e) => {
    e.preventDefault();
  };
  
  const renderColumn = (status, title, taskList) => (
    <div
      className="kanban-column"
      onDrop={(e) => handleDrop(e, status)}
      onDragOver={handleDragOver}
    >
      <h3>{title}</h3>
      <div className="task-list">
        {taskList.map(task => (
          <div
            key={task._id}
            className="task-card"
            draggable
            onDragStart={(e) => handleDragStart(e, task._id)}
          >
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <div className="task-meta">
              <span className={`priority ${task.priority}`}>
                {task.priority}
              </span>
              {task.dueDate && (
                <span className="due-date">
                  Due: {new Date(task.dueDate).toLocaleDateString()}
                </span>
              )}
            </div>
          </div>
        ))}
      </div>
    </div>
  );
  
  if (loading) return <div>Loading tasks...</div>;
  
  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do', tasks.todo)}
      {renderColumn('inProgress', 'In Progress', tasks.inProgress)}
      {renderColumn('done', 'Done', tasks.done)}
    </div>
  );
};

export default KanbanBoard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './AdminAnalytics.css';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);
  const [burnoutUsers, setBurnoutUsers] = useState([]);
  
  const API_URL = process.env.REACT_APP_API_URL;
  const ML_URL = process.env.REACT_APP_ML_SERVICE_URL;
  
  useEffect(() => {
    fetchAnalytics();
    fetchRiskUsers();
    fetchBurnoutAnalysis();
  }, []);
  
  const fetchAnalytics = async () => {
    try {
      const res = await axios.get(`${API_URL}/admin/analytics`);
      setAnalytics(res.data.analytics);
    } catch (error) {
      console.error('Fetch analytics failed:', error);
    }
  };
  
  const fetchRiskUsers = async () => {
    try {
      const res = await axios.get(`${API_URL}/admin/users/behavior`);
      const users = res.data.users;
      
      // Analyze each user with ML service
      const riskAnalysis = await Promise.all(
        users.map(async (user) => {
          try {
            const mlRes = await axios.post(`${ML_URL}/detect-risk`, {
              user_id: user._id,
              login_count: user.loginCount || 0,
              failed_logins: user.failedLogins || 0,
              avg_session_duration: user.avgSessionDuration || 0,
              unusual_access_times: user.unusualAccessTimes || 0,
              data_access_volume: user.dataAccessVolume || 0
            });
            return { ...user, risk: mlRes.data };
          } catch (error) {
            return { ...user, risk: null };
          }
        })
      );
      
      // Filter high-risk users
      const highRisk = riskAnalysis.filter(
        u => u.risk && u.risk.risk_level === 'high'
      );
      setRiskUsers(highRisk);
    } catch (error) {
      console.error('Risk analysis failed:', error);
    }
  };
  
  const fetchBurnoutAnalysis = async () => {
    try {
      const res = await axios.get(`${API_URL}/admin/users/workload`);
      const users = res.data.users;
      
      const burnoutAnalysis = await Promise.all(
        users.map(async (user) => {
          try {
            const mlRes = await axios.post(`${ML_URL}/detect-burnout`, {
              user_id: user._id,
              tasks_completed: user.tasksCompleted || 0,
              tasks_pending: user.tasksPending || 0,
              avg_task_time: user.avgTaskTime || 0,
              overtime_hours: user.overtimeHours || 0,
              missed_deadlines: user.missedDeadlines || 0
            });
            return { ...user, burnout: mlRes.data };
          } catch (error) {
            return { ...user, burnout: null };
          }
        })
      );
      
      const atRisk = burnoutAnalysis.filter(
        u => u.burnout && u.burnout.burnout_level !== 'low'
      );
      setBurnoutUsers(atRisk);
    } catch (error) {
      console.error('Burnout analysis failed:', error);
    }
  };
  
  return (
    <div className="admin-analytics">
      <h2>Organization Analytics</h2>
      
      {analytics && (
        <div className="stats-grid">
          <div className="stat-card">
            <h3>Total Users</h3>
            <p className="stat-value">{analytics.totalUsers}</p>
          </div>
          <div className="stat-card">
            <h3>Active Tasks</h3>
            <p className="stat-value">{analytics.activeTasks}</p>
          </div>
          <div className="stat-card">
            <h3>Open Tickets</h3>
            <p className="stat-value">{analytics.openTickets}</p>
          </div>
          <div className="stat-card">
            <h3>Completion Rate</h3>
            <p className="stat-value">{analytics.completionRate}%</p>
          </div>
        </div>
      )}
      
      <div className="alerts-section">
        <h3>Security Alerts</h3>
        {riskUsers.length > 0 ? (
          <div className="risk-users">
            {riskUsers.map(user => (
              <div key={user._id} className="alert-card high-risk">
                <h4>{user.name}</h4>
                <p>Risk Level: {user.risk.risk_level}</p>
                <p>Score: {user.risk.risk_score}/100</p>
                <ul>
                  {user.risk.risk_factors.map((factor, idx) => (
                    <li key={idx}>{factor}</li>
                  ))}
                </ul>
                <p className="recommendation">
                  {user.risk.recommendation}
                </p>
              </div>
            ))}
          </div>
        ) : (
          <p>No high-risk users detected</p>
        )}
      </div>
      
      <div className="burnout-section">
        <h3>Burnout Alerts</h3>
        {burnoutUsers.length > 0 ? (
          <div className="burnout-users">
            {burnoutUsers.map(user => (
              <div key={user._id} className="alert-card burnout">
                <h4>{user.name}</h4>
                <p>Burnout Level: {user.burnout.burnout_level}</p>
                <p>Score: {user.burnout.burnout_score}/100</p>
                <ul>
                  {user.burnout.indicators.map((indicator, idx) => (
                    <li key={idx}>{indicator}</li>
                  ))}
                </ul>
                <p className="recommendation">
                  {user.burnout.recommended_action}
                </p>
              </div>
            ))}
          </div>
        ) : (
          <p>No burnout risks detected</p>
        )}
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [
