---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "setup enterprise user management system"
  - "configure AI analytics for user management"
  - "implement task tracking with kanban board"
  - "add AI ticket classification"
  - "create user dashboard with analytics"
  - "setup ML service for risk prediction"
  - "integrate AI anomaly detection"
  - "build admin panel with user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics. It provides user authentication, role-based access control, task management with Kanban boards, support ticket systems, and machine learning features including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## Installation

### Prerequisites

- Node.js (v14+)
- Python 3.8+
- MongoDB

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install
```

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MONGODB_URI=your_mongodb_connection_string
MODEL_PATH=./models
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Core Components

### Authentication System

**Login Implementation (Frontend)**

```javascript
// frontend/src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const login = async (email, password) => {
  try {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    
    return response.data;
  } catch (error) {
    throw error.response.data;
  }
};

export const logout = () => {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
};

export const getCurrentUser = () => {
  return JSON.parse(localStorage.getItem('user'));
};

export const getAuthToken = () => {
  return localStorage.getItem('token');
};
```

**Protected Route Component**

```javascript
// frontend/src/components/ProtectedRoute.js
import React from 'react';
import { Navigate } from 'react-router-dom';
import { getAuthToken } from '../services/authService';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = getAuthToken();
  const user = JSON.parse(localStorage.getItem('user'));
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Backend API Endpoints

**User Management API**

```javascript
// backend/routes/userRoutes.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { authenticate, authorize } = require('../middleware/auth');

// Get all users (Admin only)
router.get('/', authenticate, authorize(['admin']), async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create user (Admin only)
router.post('/', authenticate, authorize(['admin']), async (req, res) => {
  try {
    const { name, email, role, department } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = new User({
      name,
      email,
      role,
      department,
      password: 'defaultPassword123' // Should be changed on first login
    });
    
    await user.save();
    res.status(201).json({ message: 'User created successfully', user });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update user
router.put('/:id', authenticate, authorize(['admin']), async (req, res) => {
  try {
    const { name, email, role, department, status } = req.body;
    
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { name, email, role, department, status },
      { new: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Delete user
router.delete('/:id', authenticate, authorize(['admin']), async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

**Task Management API**

```javascript
// backend/routes/taskRoutes.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authenticate } = require('../middleware/auth');

// Get user tasks
router.get('/my-tasks', authenticate, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task
router.post('/', authenticate, async (req, res) => {
  try {
    const { title, description, priority, assignedTo, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      priority,
      assignedTo,
      assignedBy: req.user.id,
      dueDate,
      status: 'todo'
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', authenticate, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Track time on task
router.post('/:id/time-log', authenticate, async (req, res) => {
  try {
    const { duration } = req.body;
    
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeSpent = (task.timeSpent || 0) + duration;
    task.timeLogs.push({
      userId: req.user.id,
      duration,
      timestamp: Date.now()
    });
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### AI/ML Service Implementation

**ML Service Main Application**

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

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketData(BaseModel):
    title: str
    description: str
    priority: str
    userId: str

class UserBehavior(BaseModel):
    userId: str
    loginTimes: List[str]
    taskCompletionRate: float
    averageResponseTime: float
    failedLogins: int

class WorkloadData(BaseModel):
    userId: str
    tasksAssigned: int
    tasksCompleted: int
    hoursWorked: float
    overtimeHours: float

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """
    Classify support ticket and route to appropriate department
    """
    try:
        # Simple rule-based classification (can be replaced with ML model)
        text = f"{ticket.title} {ticket.description}".lower()
        
        categories = {
            'technical': ['error', 'bug', 'crash', 'not working', 'issue'],
            'access': ['login', 'password', 'permission', 'access', 'authentication'],
            'feature': ['request', 'feature', 'enhancement', 'suggestion'],
            'billing': ['payment', 'invoice', 'billing', 'subscription']
        }
        
        scores = {}
        for category, keywords in categories.items():
            score = sum(1 for keyword in keywords if keyword in text)
            scores[category] = score
        
        category = max(scores, key=scores.get)
        confidence = scores[category] / len(categories[category]) if scores[category] > 0 else 0.5
        
        department_mapping = {
            'technical': 'IT Support',
            'access': 'Security',
            'feature': 'Product Team',
            'billing': 'Finance'
        }
        
        return {
            'category': category,
            'department': department_mapping[category],
            'confidence': confidence,
            'priority': calculate_priority(ticket.priority, category)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def calculate_priority(current_priority: str, category: str) -> str:
    """Calculate recommended priority based on category"""
    high_priority_categories = ['technical', 'access']
    
    if category in high_priority_categories and current_priority == 'medium':
        return 'high'
    
    return current_priority

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(behavior: UserBehavior):
    """
    Detect anomalous user behavior for security
    """
    try:
        anomaly_score = 0
        flags = []
        
        # Check failed login attempts
        if behavior.failedLogins > 5:
            anomaly_score += 30
            flags.append('High failed login attempts')
        
        # Check task completion rate
        if behavior.taskCompletionRate < 0.3:
            anomaly_score += 20
            flags.append('Low task completion rate')
        
        # Check response time
        if behavior.averageResponseTime > 48:  # hours
            anomaly_score += 15
            flags.append('Slow response time')
        
        # Check unusual login times
        unusual_hours = check_unusual_login_times(behavior.loginTimes)
        if unusual_hours > 3:
            anomaly_score += 25
            flags.append('Unusual login hours detected')
        
        is_anomaly = anomaly_score > 50
        risk_level = 'high' if anomaly_score > 70 else 'medium' if anomaly_score > 40 else 'low'
        
        return {
            'isAnomaly': is_anomaly,
            'anomalyScore': anomaly_score,
            'riskLevel': risk_level,
            'flags': flags,
            'recommendation': 'Investigate user activity' if is_anomaly else 'Normal behavior'
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def check_unusual_login_times(login_times: List[str]) -> int:
    """Check for logins during unusual hours (10 PM - 6 AM)"""
    unusual_count = 0
    
    for login_time in login_times:
        try:
            hour = datetime.fromisoformat(login_time).hour
            if hour >= 22 or hour <= 6:
                unusual_count += 1
        except:
            continue
    
    return unusual_count

@app.post("/api/ml/predict-burnout")
async def predict_burnout(workload: WorkloadData):
    """
    Predict employee burnout risk based on workload metrics
    """
    try:
        burnout_score = 0
        factors = []
        
        # Calculate task load ratio
        completion_rate = workload.tasksCompleted / workload.tasksAssigned if workload.tasksAssigned > 0 else 0
        
        if workload.tasksAssigned > 20:
            burnout_score += 25
            factors.append('High task volume')
        
        if completion_rate < 0.6:
            burnout_score += 20
            factors.append('Low completion rate - possible overload')
        
        if workload.hoursWorked > 50:
            burnout_score += 30
            factors.append('Excessive working hours')
        
        if workload.overtimeHours > 10:
            burnout_score += 25
            factors.append('High overtime hours')
        
        risk_level = 'high' if burnout_score > 60 else 'medium' if burnout_score > 35 else 'low'
        
        recommendations = []
        if burnout_score > 35:
            recommendations.append('Redistribute workload')
            recommendations.append('Schedule check-in meeting')
        if workload.overtimeHours > 10:
            recommendations.append('Limit overtime hours')
        if completion_rate < 0.6:
            recommendations.append('Review task priorities and deadlines')
        
        return {
            'burnoutScore': burnout_score,
            'riskLevel': risk_level,
            'factors': factors,
            'recommendations': recommendations,
            'metrics': {
                'completionRate': completion_rate,
                'avgHoursPerTask': workload.hoursWorked / workload.tasksAssigned if workload.tasksAssigned > 0 else 0
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/ml/health")
async def health_check():
    return {"status": "healthy", "service": "ML Service"}
```

### Frontend Dashboard Components

**User Dashboard with Task Board**

```javascript
// frontend/src/components/UserDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthToken } from '../services/authService';

const API_URL = process.env.REACT_APP_API_URL;

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [stats, setStats] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
    fetchStats();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks/my-tasks`, {
        headers: { Authorization: `Bearer ${getAuthToken()}` }
      });
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const fetchStats = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/users/stats`, {
        headers: { Authorization: `Bearer ${getAuthToken()}` }
      });
      setStats(response.data);
    } catch (error) {
      console.error('Error fetching stats:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${getAuthToken()}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task, onDragStart }) => (
    <div
      className="task-card"
      draggable
      onDragStart={(e) => onDragStart(e, task)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <span className="due-date">Due: {new Date(task.dueDate).toLocaleDateString()}</span>
      </div>
    </div>
  );

  const handleDragStart = (e, task) => {
    e.dataTransfer.setData('task', JSON.stringify(task));
  };

  const handleDrop = (e, status) => {
    e.preventDefault();
    const task = JSON.parse(e.dataTransfer.getData('task'));
    updateTaskStatus(task._id, status);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <div className="stats-overview">
        <div className="stat-card">
          <h3>Total Tasks</h3>
          <p>{stats?.totalTasks || 0}</p>
        </div>
        <div className="stat-card">
          <h3>Completed</h3>
          <p>{stats?.completedTasks || 0}</p>
        </div>
        <div className="stat-card">
          <h3>In Progress</h3>
          <p>{tasks.inProgress.length}</p>
        </div>
      </div>

      <div className="kanban-board">
        <div
          className="kanban-column"
          onDrop={(e) => handleDrop(e, 'todo')}
          onDragOver={handleDragOver}
        >
          <h3>To Do ({tasks.todo.length})</h3>
          {tasks.todo.map(task => (
            <TaskCard key={task._id} task={task} onDragStart={handleDragStart} />
          ))}
        </div>

        <div
          className="kanban-column"
          onDrop={(e) => handleDrop(e, 'in-progress')}
          onDragOver={handleDragOver}
        >
          <h3>In Progress ({tasks.inProgress.length})</h3>
          {tasks.inProgress.map(task => (
            <TaskCard key={task._id} task={task} onDragStart={handleDragStart} />
          ))}
        </div>

        <div
          className="kanban-column"
          onDrop={(e) => handleDrop(e, 'done')}
          onDragOver={handleDragOver}
        >
          <h3>Done ({tasks.done.length})</h3>
          {tasks.done.map(task => (
            <TaskCard key={task._id} task={task} onDragStart={handleDragStart} />
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

**AI Analytics Integration**

```javascript
// frontend/src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_URL = process.env.REACT_APP_ML_URL;
const API_URL = process.env.REACT_APP_API_URL;

const AIAnalytics = ({ userId }) => {
  const [burnoutRisk, setBurnoutRisk] = useState(null);
  const [anomalyDetection, setAnomalyDetection] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    try {
      // Fetch user data first
      const userDataResponse = await axios.get(`${API_URL}/api/users/${userId}/analytics`);
      const userData = userDataResponse.data;

      // Check burnout risk
      const burnoutResponse = await axios.post(`${ML_URL}/api/ml/predict-burnout`, {
        userId: userId,
        tasksAssigned: userData.tasksAssigned,
        tasksCompleted: userData.tasksCompleted,
        hoursWorked: userData.hoursWorked,
        overtimeHours: userData.overtimeHours
      });
      setBurnoutRisk(burnoutResponse.data);

      // Check for anomalies
      const anomalyResponse = await axios.post(`${ML_URL}/api/ml/detect-anomaly`, {
        userId: userId,
        loginTimes: userData.loginTimes,
        taskCompletionRate: userData.taskCompletionRate,
        averageResponseTime: userData.averageResponseTime,
        failedLogins: userData.failedLogins
      });
      setAnomalyDetection(anomalyResponse.data);

      setLoading(false);
    } catch (error) {
      console.error('Error fetching AI insights:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading AI insights...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>

      {burnoutRisk && (
        <div className={`insight-card ${burnoutRisk.riskLevel}`}>
          <h3>Burnout Risk Assessment</h3>
          <div className="risk-score">
            Risk Level: <span className={burnoutRisk.riskLevel}>{burnoutRisk.riskLevel.toUpperCase()}</span>
          </div>
          <div className="score-bar">
            <div 
              className="score-fill" 
              style={{ width: `${burnoutRisk.burnoutScore}%` }}
            />
          </div>
          <div className="factors">
            <h4>Contributing Factors:</h4>
            <ul>
              {burnoutRisk.factors.map((factor, idx) => (
                <li key={idx}>{factor}</li>
              ))}
            </ul>
          </div>
          {burnoutRisk.recommendations.length > 0 && (
            <div className="recommendations">
              <h4>Recommendations:</h4>
              <ul>
                {burnoutRisk.recommendations.map((rec, idx) => (
                  <li key={idx}>{rec}</li>
                ))}
              </ul>
            </div>
          )}
        </div>
      )}

      {anomalyDetection && anomalyDetection.isAnomaly && (
        <div className="insight-card alert">
          <h3>⚠️ Anomaly Detected</h3>
          <div className="anomaly-score">
            Anomaly Score: {anomalyDetection.anomalyScore}/100
          </div>
          <div className="flags">
            <h4>Security Flags:</h4>
            <ul>
              {anomalyDetection.flags.map((flag, idx) => (
                <li key={idx}>{flag}</li>
              ))}
            </ul>
          </div>
          <p className="recommendation">{anomalyDetection.recommendation}</p>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

### Database Models

**User Model**

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['admin', 'user', 'manager'],
    default: 'user'
  },
  department: {
    type: String,
    required: true
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: {
    type: Date
  },
  loginHistory: [{
    timestamp: Date,
    ipAddress: String
  }]
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

**Task Model**

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
    ref: 'User'
  },
  dueDate: {
    type: Date,
    required: true
  },
  timeSpent: {
    type: Number,
    default: 0
  },
  timeLogs: [{
    userId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User'
    },
    duration: Number,
    timestamp: Date
  }],
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

**Ticket Model**

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
  category: {
    type: String,
    enum: ['technical', 'access', 'feature', 'billing', 'other']
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
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  department: {
    type: String
  },
  aiClassification: {
    category: String,
    confidence: Number,
    suggestedDepartment: String
  },
  comments: [{
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User'
    },
    text: String,
    timestamp: {
      type: Date,
      default: Date.now
    }
  }],
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### Creating and Classifying a Support Ticket with AI

```javascript
// frontend/src/components/CreateTicket.js
import React, { useState } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_URL = process.env.REACT_APP_ML_URL;

const CreateTicket = () => {
  const [formData, setFormData] = useState({
    title: '',
    description: '',
    priority: 'medium'
  });
  const [aiSuggestion
